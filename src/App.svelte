<script lang="ts">
  import type {
    Line,
    BasePoint,
    Settings,
    Point,
    SequenceItem,
    Shape,
  } from "./types";
  import * as d3 from "d3";
  import {
    snapToGrid,
    gridSize,
    currentFilePath,
    isUnsaved,
    showGrid,
  } from "./stores";
  import Two from "two.js";
  import type { Path } from "two.js/src/path";
  import type { Line as PathLine } from "two.js/src/shapes/line";
  import ControlTab from "./lib/ControlTab.svelte";
  import Navbar from "./lib/Navbar.svelte";
  import MathTools from "./lib/MathTools.svelte";
  import _ from "lodash";
  import hotkeys from "hotkeys-js";
  import { createAnimationController } from "./utils/animation";
  import { calculatePathTime, getAnimationDuration } from "./utils";

  import {
    calculateRobotState,
    generateGhostPathPoints,
    generateOnionLayers,
  } from "./utils";
  import {
    easeInOutQuad,
    getCurvePoint,
    getRandomColor,
    quadraticToCubic,
    radiansToDegrees,
    shortestRotation,
    downloadTrajectory,
    loadTrajectoryFromFile,
    loadRobotImage,
    updateRobotImageDisplay,
  } from "./utils";
  import {
    POINT_RADIUS,
    LINE_WIDTH,
    DEFAULT_ROBOT_WIDTH,
    DEFAULT_ROBOT_HEIGHT,
    DEFAULT_SETTINGS,
    FIELD_SIZE,
    getDefaultStartPoint,
    getDefaultLines,
    getDefaultShapes,
    RED_ALLIANCE_RED_GOAL,
    BLUE_ALLIANCE_BLUE_GOAL,
  } from "./config";
  import { loadSettings, saveSettings } from "./utils/settingsPersistence";
  import * as browserFileStore from "./utils/browserFileStore";
  import { onMount, tick } from "svelte";
  import { debounce } from "lodash";
  import { createHistory, type AppState } from "./utils/history";
  // Browser-only build: file operations use the browser file store and
  // localStorage. Electron-specific APIs have been removed.

  // Resolve asset base from current page so assets work on any base path (e.g. /Visualizer/)
  function assetUrl(path: string): string {
    const p = (path || "").replace(/^\//, "");
    if (typeof document === "undefined") return (import.meta.env.BASE_URL ?? "/") + p;
    const base = new URL(".", document.baseURI).pathname.replace(/\/?$/, "/");
    return base + p;
  }

  function normalizeLines(input: Line[]): Line[] {
    return (input || []).map((line) => ({
      ...line,
      id: line.id || `line-${Math.random().toString(36).slice(2)}`,
      controlPoints: line.controlPoints || [],
      color: line.color || getRandomColor(),
      name: line.name || "",
      waitBeforeMs: Math.max(
        0,
        Number(line.waitBeforeMs ?? line.waitBefore?.durationMs ?? 0),
      ),
      waitAfterMs: Math.max(
        0,
        Number(line.waitAfterMs ?? line.waitAfter?.durationMs ?? 0),
      ),
      waitBeforeName: line.waitBeforeName ?? line.waitBefore?.name ?? "",
      waitAfterName: line.waitAfterName ?? line.waitAfter?.name ?? "",
    }));
  }

  // Canvas state
  let two: Two;
  let twoElement: HTMLDivElement;
  let width = 0;
  let height = 0;
  // Robot state
  $: robotWidth = settings?.rWidth || DEFAULT_ROBOT_WIDTH;
  $: robotHeight = settings?.rHeight || DEFAULT_ROBOT_HEIGHT;
  let robotXY: BasePoint = { x: 0, y: 0 };
  let robotHeading: number = 0;
  // Manual bot position (field inches). When set, arrow keys have moved the bot; Escape clears.
  let manualBotPosition: { x: number; y: number } | null = null;
  /** When set, overrides animation heading (e.g. after a/d/s rotation). Cleared with Escape. */
  let manualHeading: number | null = null;
  /** When true, θ_b triangle and bot-to-goal line use turret center instead of bot center. Toggle with key "t". */
  let useTurretCenterForThetaB = false;
  /** When true, use red alliance red goal; when false, use blue alliance blue goal. */
  let useRedGoal = true;
  /** Turret angle in bot frame: 0–190°, center (forward) = 95°. q/e rotate, w recenter. */
  const TURRET_ANGLE_MIN = 0;
  const TURRET_ANGLE_MAX = 190;
  const TURRET_ANGLE_CENTER = 95;
  let turretAngle = TURRET_ANGLE_CENTER;
  let animationRobotXY: BasePoint = { x: 0, y: 0 };
  let animationRobotHeading: number = 0;
  const ARROW_KEY_STEP_INCHES = 2;
  const ROTATION_STEP_DEGREES = 1;

  function clampToField(pos: { x: number; y: number }) {
    return {
      x: Math.max(0, Math.min(FIELD_SIZE, pos.x)),
      y: Math.max(0, Math.min(FIELD_SIZE, pos.y)),
    };
  }
  // Animation state
  let percent: number = 0;
  let playing = false;
  let animationFrame: number;
  let startTime: number | null = null;
  let previousTime: number | null = null;
  // Path data
  let settings: Settings = { ...DEFAULT_SETTINGS };
  let startPoint: Point = getDefaultStartPoint();
  let lines: Line[] = normalizeLines(getDefaultLines());
  let sequence: SequenceItem[] = lines.map((ln) => ({
    kind: "path",
    lineId: ln.id!,
  }));
  let shapes: Shape[] = getDefaultShapes();
  let optimizingLineIds: Record<string, boolean> = {};
  let optimizingAll = false;

  const history = createHistory();
  const { canUndoStore, canRedoStore } = history;
  const OPTIMIZER_BASE_URL = "https://fpa.pedropathing.com";

  function getAppState(): AppState {
    return {
      startPoint,
      lines,
      shapes,
      sequence,
      settings,
    };
  }

  // Use the stores for reactivity
  $: canUndo = $canUndoStore;
  $: canRedo = $canRedoStore;

  function recordChange() {
    history.record(getAppState());
  }

  function undoAction() {
    const prev = history.undo();
    if (prev) {
      startPoint = prev.startPoint;
      lines = prev.lines;
      shapes = prev.shapes;
      sequence = prev.sequence;
      settings = prev.settings;
      isUnsaved.set(true);
      two && two.update();
    }

    // undoAction completes; no file-picker behavior here
  }

  function redoAction() {
    const next = history.redo();
    if (next) {
      startPoint = next.startPoint;
      lines = next.lines;
      shapes = next.shapes;
      sequence = next.sequence;
      settings = next.settings;
      isUnsaved.set(true);
      two && two.update();
    }
  }

  $: {
    // Ensure arrays are reactive when items are added/removed
    lines = lines;
    shapes = shapes;
  }

  // Two.js groups
  let lineGroup = new Two.Group();
  lineGroup.id = "line-group";
  let pointGroup = new Two.Group();
  pointGroup.id = "point-group";
  let shapeGroup = new Two.Group();
  shapeGroup.id = "shape-group";
  // Coordinate converters
  let x: d3.ScaleLinear<number, number, number>;

  // Animation controller
  let loopAnimation = true;
  let animationController: ReturnType<typeof createAnimationController>;
  $: timePrediction = calculatePathTime(startPoint, lines, settings, sequence);
  $: animationDuration = getAnimationDuration(timePrediction.totalTime / 1000);
  /**
   * Converter for X axis from inches to pixels.
   */
  $: x = d3
    .scaleLinear()
    .domain([0, FIELD_SIZE])
    .range([0, width || FIELD_SIZE]);
  /**
   * Converter for Y axis from inches to pixels.
   */
  $: y = d3
    .scaleLinear()
    .domain([0, FIELD_SIZE])
    .range([height || FIELD_SIZE, 0]);
  $: {
    // Calculate robot state using the Timeline (animation-derived position)
    if (timePrediction && timePrediction.timeline && lines.length > 0) {
      const state = calculateRobotState(
        percent,
        timePrediction.timeline,
        lines,
        startPoint,
        settings,
        x,
        y,
      );
      animationRobotXY = { x: state.x, y: state.y };
      animationRobotHeading = state.heading;
    } else {
      animationRobotXY = { x: x(startPoint.x), y: y(startPoint.y) };
      animationRobotHeading = 0;
    }
  }
  $: {
    // Use manual position/heading if set (clamped to field), otherwise animation
    if (manualBotPosition != null) {
      const clamped = clampToField(manualBotPosition);
      robotXY = { x: x(clamped.x), y: y(clamped.y) };
      robotHeading = manualHeading != null ? manualHeading : animationRobotHeading;
    } else {
      robotXY = animationRobotXY;
      robotHeading = animationRobotHeading;
      manualHeading = null;
    }
  }

  $: points = (() => {
    let _points = [];
    let startPointElem = new Two.Circle(
      x(startPoint.x),
      y(startPoint.y),
      x(POINT_RADIUS),
    );
    startPointElem.id = `point-0-0`;
    startPointElem.fill = lines[0].color;
    startPointElem.noStroke();

    _points.push(startPointElem);

    lines.forEach((line, idx) => {
      if (!line || !line.endPoint) return; // Skip invalid lines or lines without endPoint
      [line.endPoint, ...line.controlPoints].forEach((point, idx1) => {
        if (idx1 > 0) {
          let pointGroup = new Two.Group();
          pointGroup.id = `point-${idx + 1}-${idx1}`;

          let pointElem = new Two.Circle(
            x(point.x),
            y(point.y),
            x(POINT_RADIUS),
          );
          pointElem.id = `point-${idx + 1}-${idx1}-background`;
          pointElem.fill = line.color;
          pointElem.noStroke();

          let pointText = new Two.Text(
            `${idx1}`,
            x(point.x),
            y(point.y - 0.15),
            x(POINT_RADIUS),
          );
          pointText.id = `point-${idx + 1}-${idx1}-text`;
          pointText.size = x(1.55);
          pointText.leading = 1;
          pointText.family = "ui-sans-serif, system-ui, sans-serif";
          pointText.alignment = "center";
          pointText.baseline = "middle";
          pointText.fill = "white";
          pointText.noStroke();

          pointGroup.add(pointElem, pointText);
          _points.push(pointGroup);
        } else {
          let pointElem = new Two.Circle(
            x(point.x),
            y(point.y),
            x(POINT_RADIUS),
          );
          pointElem.id = `point-${idx + 1}-${idx1}`;
          pointElem.fill = line.color;
          pointElem.noStroke();
          _points.push(pointElem);
        }
      });
    });
    // Add obstacle vertices as draggable points
    shapes.forEach((shape, shapeIdx) => {
      shape.vertices.forEach((vertex, vertexIdx) => {
        let pointGroup = new Two.Group();
        pointGroup.id = `obstacle-${shapeIdx}-${vertexIdx}`;

        let pointElem = new Two.Circle(
          x(vertex.x),
          y(vertex.y),
          x(POINT_RADIUS),
        );
        pointElem.id = `obstacle-${shapeIdx}-${vertexIdx}-background`;
        pointElem.fill = "#991b1b"; // Match obstacle color
        pointElem.noStroke();

        let pointText = new Two.Text(
          `${vertexIdx + 1}`,
          x(vertex.x),
          y(vertex.y - 0.15),
          x(POINT_RADIUS),
        );
        pointText.id = `obstacle-${shapeIdx}-${vertexIdx}-text`;
        pointText.size = x(1.55);
        pointText.leading = 1;
        pointText.family = "ui-sans-serif, system-ui, sans-serif";
        pointText.alignment = "center";
        pointText.baseline = "middle";
        pointText.fill = "white";
        pointText.noStroke();
        pointGroup.add(pointElem, pointText);
        _points.push(pointGroup);
      });
    });

    return _points;
  })();

  $: path = (() => {
    let _path: (Path | PathLine)[] = [];

    lines.forEach((line, idx) => {
      if (!line || !line.endPoint) return; // Skip invalid lines or lines without endPoint
      let _startPoint =
        idx === 0 ? startPoint : lines[idx - 1]?.endPoint || null;
      if (!_startPoint) return; // Skip if previous line's endPoint is missing

      let lineElem: Path | PathLine;
      if (line.controlPoints.length > 2) {
        // Approximate an n-degree bezier curve by sampling it at 100 points
        const samples = 100;
        const cps = [_startPoint, ...line.controlPoints, line.endPoint];
        let points = [
          new Two.Anchor(
            x(_startPoint.x),
            y(_startPoint.y),
            0,
            0,
            0,
            0,
            Two.Commands.move,
          ),
        ];
        for (let i = 1; i <= samples; ++i) {
          const point = getCurvePoint(i / samples, cps);
          points.push(
            new Two.Anchor(
              x(point.x),
              y(point.y),
              0,
              0,
              0,
              0,
              Two.Commands.line,
            ),
          );
        }
        points.forEach((point) => (point.relative = false));
        lineElem = new Two.Path(points);
        lineElem.automatic = false;
      } else if (line.controlPoints.length > 0) {
        let cp1 = line.controlPoints[1]
          ? line.controlPoints[0]
          : quadraticToCubic(_startPoint, line.controlPoints[0], line.endPoint)
              .Q1;
        let cp2 =
          line.controlPoints[1] ??
          quadraticToCubic(_startPoint, line.controlPoints[0], line.endPoint)
            .Q2;
        let points = [
          new Two.Anchor(
            x(_startPoint.x),
            y(_startPoint.y),
            x(_startPoint.x),
            y(_startPoint.y),
            x(cp1.x),
            y(cp1.y),
            Two.Commands.move,
          ),
          new Two.Anchor(
            x(line.endPoint.x),
            y(line.endPoint.y),
            x(cp2.x),
            y(cp2.y),
            x(line.endPoint.x),
            y(line.endPoint.y),
            Two.Commands.curve,
          ),
        ];
        points.forEach((point) => (point.relative = false));

        lineElem = new Two.Path(points);
        lineElem.automatic = false;
      } else {
        lineElem = new Two.Line(
          x(_startPoint.x),
          y(_startPoint.y),
          x(line.endPoint.x),
          y(line.endPoint.y),
        );
      }

      lineElem.id = `line-${idx + 1}`;
      lineElem.stroke = line.color;
      lineElem.linewidth = x(LINE_WIDTH);
      lineElem.noFill();
      // Add a dashed line for locked paths
      if (line.locked) {
        lineElem.dashes = [x(2), x(2)];
        lineElem.opacity = 0.7;
      } else {
        lineElem.dashes = [];
        lineElem.opacity = 1;
      }

      _path.push(lineElem);
    });

    return _path;
  })();
  $: shapeElements = (() => {
    // Obstacles removed: return empty array for shape elements
    let _shapes: Path[] = [];

    shapes.forEach((shape, idx) => {
      if (shape.vertices.length >= 3) {
        // Create polygon from vertices - properly format for Two.js
        let vertices = [];

        // Start with move command for first vertex
        vertices.push(
          new Two.Anchor(
            x(shape.vertices[0].x),
            y(shape.vertices[0].y),
            0,
            0,
            0,
            0,
            Two.Commands.move,
          ),
        );

        // Add line commands for remaining vertices
        for (let i = 1; i < shape.vertices.length; i++) {
          vertices.push(
            new Two.Anchor(
              x(shape.vertices[i].x),
              y(shape.vertices[i].y),
              0,
              0,
              0,
              0,
              Two.Commands.line,
            ),
          );
        }

        // Close the shape
        vertices.push(
          new Two.Anchor(
            x(shape.vertices[0].x),
            y(shape.vertices[0].y),
            0,
            0,
            0,
            0,
            Two.Commands.close,
          ),
        );

        vertices.forEach((point) => (point.relative = false));

        let shapeElement = new Two.Path(vertices);
        shapeElement.id = `shape-${idx}`;
        shapeElement.stroke = shape.color;
        shapeElement.fill = shape.color;
        shapeElement.opacity = 0.4;
        shapeElement.linewidth = x(0.8);
        shapeElement.automatic = false;

        _shapes.push(shapeElement);
      }
    });

    return _shapes;
  })();

  $: ghostPathElement = (() => {
    let ghostPath: Path | null = null;

    if (settings.showGhostPaths && lines.length > 0) {
      const ghostPoints = generateGhostPathPoints(
        startPoint,
        lines,
        settings.rWidth,
        settings.rHeight,
        50,
      );

      if (ghostPoints.length >= 3) {
        // Create polygon from ghost path points
        let vertices = [];

        // Start with move command for first point
        vertices.push(
          new Two.Anchor(
            x(ghostPoints[0].x),
            y(ghostPoints[0].y),
            0,
            0,
            0,
            0,
            Two.Commands.move,
          ),
        );

        // Add line commands for remaining points
        for (let i = 1; i < ghostPoints.length; i++) {
          vertices.push(
            new Two.Anchor(
              x(ghostPoints[i].x),
              y(ghostPoints[i].y),
              0,
              0,
              0,
              0,
              Two.Commands.line,
            ),
          );
        }

        // Close the shape
        vertices.push(
          new Two.Anchor(
            x(ghostPoints[0].x),
            y(ghostPoints[0].y),
            0,
            0,
            0,
            0,
            Two.Commands.close,
          ),
        );

        vertices.forEach((point) => (point.relative = false));

        ghostPath = new Two.Path(vertices);
        ghostPath.id = "ghost-path";
        ghostPath.stroke = "#a78bfa"; // Light purple/lavender
        ghostPath.fill = "#a78bfa";
        ghostPath.opacity = 0.15;
        ghostPath.linewidth = x(0.5);
        ghostPath.automatic = false;
      }
    }

    return ghostPath;
  })();

  $: onionLayerElements = (() => {
    let onionLayers: Path[] = [];

    if (settings.showOnionLayers && lines.length > 0) {
      const spacing = settings.onionLayerSpacing || 6;
      let layers = generateOnionLayers(
        startPoint,
        lines,
        settings.rWidth,
        settings.rHeight,
        spacing,
      );

      // If user requested onion layers only for the next point, filter to the relevant line
      if (
        settings.onionNextPointOnly &&
        timePrediction &&
        timePrediction.timeline
      ) {
        const currentTime = (timePrediction.totalTime || 0) * (percent / 100);
        const travelEvents = (timePrediction.timeline || []).filter(
          (ev) => ev.type === "travel",
        );

        let selectedLineIndex: number | null = null;

        // Current travel segment
        const currentTravel = travelEvents.find(
          (ev) => ev.startTime <= currentTime && ev.endTime >= currentTime,
        );
        if (currentTravel) {
          selectedLineIndex = currentTravel.lineIndex as number;
        } else {
          // Next upcoming travel segment
          const nextTravel = travelEvents.find(
            (ev) => ev.startTime > currentTime,
          );
          if (nextTravel) selectedLineIndex = nextTravel.lineIndex as number;
          else if (travelEvents.length)
            selectedLineIndex = travelEvents[travelEvents.length - 1]
              .lineIndex as number;
        }

        if (selectedLineIndex !== null) {
          layers = layers.filter((l: any) => l.lineIndex === selectedLineIndex);
        }
      }

      layers.forEach((layer, idx) => {
        // Create a rectangle from the robot corners
        let vertices: any[] = [];

        // Create path from corners: front-left -> front-right -> back-right -> back-left
        vertices.push(
          new Two.Anchor(
            x(layer.corners[0].x),
            y(layer.corners[0].y),
            0,
            0,
            0,
            0,
            Two.Commands.move,
          ),
        );

        for (let i = 1; i < layer.corners.length; i++) {
          vertices.push(
            new Two.Anchor(
              x(layer.corners[i].x),
              y(layer.corners[i].y),
              0,
              0,
              0,
              0,
              Two.Commands.line,
            ),
          );
        }

        // Close the path by returning to the first corner
        vertices.push(
          new Two.Anchor(
            x(layer.corners[0].x),
            y(layer.corners[0].y),
            0,
            0,
            0,
            0,
            Two.Commands.close,
          ),
        );

        vertices.forEach((point) => (point.relative = false));

        let onionRect = new Two.Path(vertices);
        onionRect.id = `onion-layer-${idx}`;
        onionRect.stroke = settings.onionColor || "#dc2626";
        onionRect.noFill();
        // Increase opacity so colliders are more visible
        onionRect.opacity = 0.9;
        onionRect.linewidth = x(0.28);
        onionRect.automatic = false;

        onionLayers.push(onionRect);
      });
    }

    return onionLayers;
  })();

  // Goal position: red or blue alliance goal (toggle in UI).
  $: goalCenter = useRedGoal ? RED_ALLIANCE_RED_GOAL : BLUE_ALLIANCE_BLUE_GOAL;
  /** When false: snap to tag (θ_b line to goal center). When true: snap to goal (line extended 6" in +x). */
  let snapToGoal = false;
  $: thetaBTarget = snapToGoal
    ? {
        x: goalCenter.x + (useRedGoal ? 6 : -6),
        y: goalCenter.y + 6,
      }
    : goalCenter;

  // Turret center in pixels (same offset as when θ_b uses turret: 25% robot height toward robot's left).
  $: turretCenterPx = (() => {
    const rad = (robotHeading * Math.PI) / 180;
    const offsetPx = x(robotHeight) / 4;
    return {
      x: robotXY.x + offsetPx * Math.sin(rad),
      y: robotXY.y - offsetPx * Math.cos(rad),
    };
  })();

  // Origin for θ_b (bot or turret). Turret offset: 25% of robot height toward robot's left.
  $: thetaBOriginPx = useTurretCenterForThetaB ? turretCenterPx : robotXY;

  $: botToGoalElements = (() => {
    const gx = x(thetaBTarget.x);
    const gy = y(thetaBTarget.y);
    const xb = x.invert(thetaBOriginPx.x);
    const yb = y.invert(thetaBOriginPx.y);
    const xg = thetaBTarget.x;
    const yg = thetaBTarget.y;
    const theta_b_rad = Math.atan2(yg - yb, xg - xb);
    const theta_b_deg = (theta_b_rad * 180) / Math.PI;

    // θ_b color: red for red goal, blue for blue goal
    const goalStroke = useRedGoal ? "#dc2626" : "#2563eb";
    const goalStrokeDark = useRedGoal ? "#b91c1c" : "#1d4ed8";

    // θ_b arc: from field +X (0) to origin→goal direction, so the computed angle is visible
    const thetaBArcRadiusPx = x(3);
    const thetaBArcEndAngle = Math.atan2(
      gy - thetaBOriginPx.y,
      gx - thetaBOriginPx.x,
    );
    const thetaBArc = new Two.ArcSegment(
      thetaBOriginPx.x,
      thetaBOriginPx.y,
      0,
      thetaBArcRadiusPx,
      0,
      thetaBArcEndAngle,
      32,
    );
    thetaBArc.noFill();
    thetaBArc.stroke = goalStroke;
    thetaBArc.linewidth = Math.max(4, x(0.5));
    thetaBArc.opacity = 0.9;

    // Heading: 12-inch arrow from robot center only (unchanged by turret toggle)
    const xb_robot = x.invert(robotXY.x);
    const yb_robot = y.invert(robotXY.y);
    const HEADING_ARROW_INCHES = 12;
    const heading_rad = (-robotHeading * Math.PI) / 180;
    const ch = Math.cos(heading_rad);
    const sh = Math.sin(heading_rad);
    const tipX_field = xb_robot + HEADING_ARROW_INCHES * ch;
    const tipY_field = yb_robot + HEADING_ARROW_INCHES * sh;
    const tipPx = x(tipX_field);
    const tipPy = y(tipY_field);

    const headingArrowLine = new Two.Line(robotXY.x, robotXY.y, tipPx, tipPy);
    headingArrowLine.stroke = "#16a34a";
    headingArrowLine.linewidth = Math.max(2, x(0.2));
    headingArrowLine.noFill();

    const arrDx = tipPx - robotXY.x;
    const arrDy = tipPy - robotXY.y;
    const arrLen = Math.sqrt(arrDx * arrDx + arrDy * arrDy) || 1;
    const arrUx = arrDx / arrLen;
    const arrUy = arrDy / arrLen;
    const arrHeadLen = Math.max(10, x(1));
    const arrHeadW = Math.max(5, x(0.5));
    const headBack1X = tipPx - arrUx * arrHeadLen + arrUy * arrHeadW;
    const headBack1Y = tipPy - arrUy * arrHeadLen - arrUx * arrHeadW;
    const headBack2X = tipPx - arrUx * arrHeadLen - arrUy * arrHeadW;
    const headBack2Y = tipPy - arrUy * arrHeadLen + arrUx * arrHeadW;
    const headingArrowHead = new Two.Path(
      [
        new Two.Anchor(tipPx, tipPy, 0, 0, 0, 0, Two.Commands.move),
        new Two.Anchor(headBack1X, headBack1Y, 0, 0, 0, 0, Two.Commands.line),
        new Two.Anchor(headBack2X, headBack2Y, 0, 0, 0, 0, Two.Commands.line),
        new Two.Anchor(tipPx, tipPy, 0, 0, 0, 0, Two.Commands.close),
      ],
      true,
    );
    headingArrowHead.fill = "#16a34a";
    headingArrowHead.stroke = "#15803d";
    headingArrowHead.linewidth = 1;

    const arcRadiusPx = x(4);
    const arcStartAngle = 0;
    const arcEndAngle = Math.atan2(-sh, ch);
    const headingArc = new Two.ArcSegment(
      robotXY.x,
      robotXY.y,
      0,
      arcRadiusPx,
      arcStartAngle,
      arcEndAngle,
      32,
    );
    headingArc.noFill();
    headingArc.stroke = "#16a34a";
    headingArc.linewidth = Math.max(4, x(0.5));

    const theta_h_deg = -robotHeading;
    const arcMidAngle = (arcStartAngle + arcEndAngle) / 2;
    const labelRadius = arcRadiusPx * 1.4;
    const headingLabelOffsetRight = x(4.5);
    const headingLabelX = robotXY.x + Math.cos(arcMidAngle) * labelRadius + headingLabelOffsetRight;
    const headingLabelY = robotXY.y - Math.sin(arcMidAngle) * labelRadius;
    const angleFontSize = Math.max(18, x(3));
    const headingLabel = new Two.Text(
      `θₕ = ${theta_h_deg.toFixed(1)}°`,
      headingLabelX,
      headingLabelY,
      { size: angleFontSize, leading: angleFontSize },
    );
    headingLabel.fill = "#15803d";
    headingLabel.family = "ui-sans-serif, system-ui, sans-serif";
    headingLabel.weight = "700";
    headingLabel.alignment = "center";
    headingLabel.baseline = "middle";
    headingLabel.style = "user-select: none;";

    // θ_t: turret angle (white). W.r.t. bot: 0–190°, center 95° = forward. Direction = robotHeading + (turretAngle - 95).
    const turretDirInFieldDeg = robotHeading + (turretAngle - TURRET_ANGLE_CENTER);
    const turret_rad = (-turretDirInFieldDeg * Math.PI) / 180;
    const ct = Math.cos(turret_rad);
    const st = Math.sin(turret_rad);
    const TURRET_ARROW_INCHES = 6;
    const turretTipX_field = x.invert(turretCenterPx.x) + TURRET_ARROW_INCHES * ct;
    const turretTipY_field = y.invert(turretCenterPx.y) + TURRET_ARROW_INCHES * st;
    const turretTipPx = x(turretTipX_field);
    const turretTipPy = y(turretTipY_field);
    const turretArrowLine = new Two.Line(turretCenterPx.x, turretCenterPx.y, turretTipPx, turretTipPy);
    turretArrowLine.stroke = "#ffffff";
    turretArrowLine.linewidth = Math.max(1.5, x(0.15));
    turretArrowLine.noFill();
    const tArrDx = turretTipPx - turretCenterPx.x;
    const tArrDy = turretTipPy - turretCenterPx.y;
    const tArrLen = Math.sqrt(tArrDx * tArrDx + tArrDy * tArrDy) || 1;
    const tArrUx = tArrDx / tArrLen;
    const tArrUy = tArrDy / tArrLen;
    const tHeadLen = Math.max(6, x(0.6));
    const tHeadW = Math.max(3, x(0.3));
    const tBack1X = turretTipPx - tArrUx * tHeadLen + tArrUy * tHeadW;
    const tBack1Y = turretTipPy - tArrUy * tHeadLen - tArrUx * tHeadW;
    const tBack2X = turretTipPx - tArrUx * tHeadLen - tArrUy * tHeadW;
    const tBack2Y = turretTipPy - tArrUy * tHeadLen + tArrUx * tHeadW;
    const turretArrowHead = new Two.Path(
      [
        new Two.Anchor(turretTipPx, turretTipPy, 0, 0, 0, 0, Two.Commands.move),
        new Two.Anchor(tBack1X, tBack1Y, 0, 0, 0, 0, Two.Commands.line),
        new Two.Anchor(tBack2X, tBack2Y, 0, 0, 0, 0, Two.Commands.line),
        new Two.Anchor(turretTipPx, turretTipPy, 0, 0, 0, 0, Two.Commands.close),
      ],
      true,
    );
    turretArrowHead.fill = "#ffffff";
    turretArrowHead.stroke = "#e5e5e5";
    turretArrowHead.linewidth = 1;
    const turretArcRadiusPx = x(2.5);
    const turretArcStart = Math.atan2(-sh, ch)-1.66;
    const turretArcEnd = Math.atan2(-st, ct);
    const turretArc = new Two.ArcSegment(
      turretCenterPx.x,
      turretCenterPx.y,
      0,
      turretArcRadiusPx,
      turretArcStart,
      turretArcEnd,
      24,
    );
    turretArc.noFill();
    turretArc.stroke = "#ffffff";
    turretArc.linewidth = Math.max(1.5, x(0.2));
    const theta_t_deg = turretAngle;
    const turretArcMid = (turretArcStart + turretArcEnd) / 2;
    const turretLabelRadius = turretArcRadiusPx * 1.3;
    const turretLabelX = turretCenterPx.x + Math.cos(turretArcMid) * turretLabelRadius;
    const turretLabelY = turretCenterPx.y - Math.sin(turretArcMid) * turretLabelRadius;
    const turretFontSize = Math.max(14, x(2.2));
    const turretLabel = new Two.Text(
      `θₜ = ${theta_t_deg.toFixed(1)}°`,
      turretLabelX,
      turretLabelY,
      { size: turretFontSize, leading: turretFontSize },
    );
    turretLabel.fill = "#ffffff";
    turretLabel.family = "ui-sans-serif, system-ui, sans-serif";
    turretLabel.weight = "700";
    turretLabel.alignment = "center";
    turretLabel.baseline = "middle";
    turretLabel.style = "user-select: none;";

    // Label θ_b along the arrow (origin → goal); b as subscript
    const labelX = thetaBOriginPx.x + (gx - thetaBOriginPx.x) * 0.28;
    const labelY = thetaBOriginPx.y + (gy - thetaBOriginPx.y) * 0.12;
    const thetaLabelGroup = new Two.Group();
    thetaLabelGroup.position.set(labelX, labelY);
    const thetaSym = new Two.Text("θ", 0, 0, { size: angleFontSize, leading: angleFontSize });
    thetaSym.fill = goalStrokeDark;
    thetaSym.family = "ui-sans-serif, system-ui, sans-serif";
    thetaSym.weight = "700";
    thetaSym.alignment = "left";
    thetaSym.baseline = "middle";
    thetaSym.style = "user-select: none;";
    const subB = new Two.Text("b", angleFontSize * 0.55, angleFontSize * 0.2, { size: angleFontSize * 0.65, leading: angleFontSize * 0.65 });
    subB.fill = goalStrokeDark;
    subB.family = "ui-sans-serif, system-ui, sans-serif";
    subB.weight = "700";
    subB.alignment = "left";
    subB.baseline = "middle";
    subB.style = "user-select: none;";
    const thetaValue = new Two.Text(` = ${theta_b_deg.toFixed(1)}°`, angleFontSize * 0.9, 0, { size: angleFontSize, leading: angleFontSize });
    thetaValue.fill = goalStrokeDark;
    thetaValue.family = "ui-sans-serif, system-ui, sans-serif";
    thetaValue.weight = "700";
    thetaValue.alignment = "left";
    thetaValue.baseline = "middle";
    thetaValue.style = "user-select: none;";
    thetaLabelGroup.add(thetaSym, subB, thetaValue);

    const line = new Two.Line(thetaBOriginPx.x, thetaBOriginPx.y, gx, gy);
    line.stroke = goalStroke;
    line.linewidth = Math.max(1.5, x(0.2));
    line.noFill();

    const dx = thetaBOriginPx.x - gx;
    const dy = thetaBOriginPx.y - gy;
    const len = Math.sqrt(dx * dx + dy * dy) || 1;
    const ux = dx / len;
    const uy = dy / len;
    const perpX = -uy;
    const perpY = ux;
    const arrowLen = Math.max(12, x(1.2));
    const arrowWidth = Math.max(6, x(0.6));
    const tipX = gx;
    const tipY = gy;
    const back1X = gx + ux * arrowLen + perpX * arrowWidth;
    const back1Y = gy + uy * arrowLen + perpY * arrowWidth;
    const back2X = gx + ux * arrowLen - perpX * arrowWidth;
    const back2Y = gy + uy * arrowLen - perpY * arrowWidth;
    const arrowHead = new Two.Path(
      [
        new Two.Anchor(tipX, tipY, 0, 0, 0, 0, Two.Commands.move),
        new Two.Anchor(back1X, back1Y, 0, 0, 0, 0, Two.Commands.line),
        new Two.Anchor(back2X, back2Y, 0, 0, 0, 0, Two.Commands.line),
        new Two.Anchor(tipX, tipY, 0, 0, 0, 0, Two.Commands.close),
      ],
      true,
    );
    arrowHead.fill = goalStroke;
    arrowHead.stroke = goalStrokeDark;
    arrowHead.linewidth = 1;
    arrowHead.opacity = 0.9;
    // Draw heading triangle and label on top so they aren’t covered by the red θ_b triangle
    // Z-order: red θ_b arrow + arc + label, then green θ_h / white θ_t on top
    return {
      elements: [
        line,
        arrowHead,
        thetaBArc,
        thetaLabelGroup,
        headingArc,
        headingArrowLine,
        headingArrowHead,
        headingLabel,
        turretArc,
        turretArrowLine,
        turretArrowHead,
        turretLabel,
      ],
    };
  })();

  let isLoaded = false;
  // Reactively trigger when any saveable data changes
  $: {
    if (isLoaded && (lines || shapes || startPoint || settings)) {
      isUnsaved.set(true);
    }
  }

  // Allow the app to stabilize before tracking changes
  onMount(() => {
    setTimeout(() => {
      isLoaded = true;
      recordChange();
    }, 500);
  });
  onMount(async () => {
    // Load saved settings
    const savedSettings = await loadSettings();
    settings = { ...savedSettings };

    // Update robot dimensions from loaded settings
    robotWidth = settings.rWidth;
    robotHeight = settings.rHeight;
  });
  // Debounced save function
  const debouncedSaveSettings = debounce(async (settingsToSave: Settings) => {
    await saveSettings(settingsToSave);
  }, 1000);
  // Save after 1 second of inactivity

  // Watch for settings changes and save
  $: {
    if (settings) {
      debouncedSaveSettings(settings);
    }
  }

  // Initialize animation controller
  onMount(() => {
    animationController = createAnimationController(
      animationDuration,
      (newPercent) => {
        percent = newPercent;
      },
      () => {
        // Animation completed callback
        console.log("Animation completed");
        playing = false;
      },
    );
  });
  $: if (animationController) {
    animationController.setDuration(animationDuration);
  }

  $: if (animationController) {
    animationController.setLoop(loopAnimation);
    // Sync UI state with controller
    playing = animationController.isPlaying();
  }

  // Save Function
  // Save the current project into the browser-backed store (or download)
  async function saveProject() {
    try {
      await saveFile();
    } catch (e) {
      console.error("Failed to save project:", e);
      alert("Failed to save file.");
    }
  }

  // Keyboard shortcut for save
  hotkeys("cmd+s, ctrl+s", function (event, handler) {
    event.preventDefault();
    saveProject();
  });

  // Event markers removed: no runtime visualization created

  $: (() => {
    if (!two) {
      return;
    }

    two.renderer.domElement.style["z-index"] = "45";
    two.renderer.domElement.style["position"] = "absolute";
    two.renderer.domElement.style["top"] = "0px";
    two.renderer.domElement.style["left"] = "0px";
    two.renderer.domElement.style["width"] = "100%";
    two.renderer.domElement.style["height"] = "100%";

    two.clear();

    two.add(...shapeElements);
    if (ghostPathElement) {
      two.add(ghostPathElement);
    }
    if (onionLayerElements.length > 0) {
      two.add(...onionLayerElements);
    }
    // Path building disabled: only move bot & turret mode; paths not drawn
    two.add(...botToGoalElements.elements);

    two.update();
  })();
  async function saveFileAs() {
    const win: any = window as any;
    const content = JSON.stringify(
      {
        startPoint,
        lines,
        shapes,
        sequence,
        settings,
        version: "1.2.1",
        timestamp: new Date().toISOString(),
      },
      null,
      2,
    );

    // Prefer File System Access API if available: opens native Save dialog
    if (win.showSaveFilePicker) {
      try {
        const opts = {
          suggestedName: $currentFilePath
            ? $currentFilePath.split(/[\/]/).pop()
            : "path.pp",
          types: [
            {
              description: "Path files",
              accept: { "application/json": [".pp", ".json"] },
            },
          ],
        };

        const handle = await win.showSaveFilePicker(opts);
        if (!handle) {
          // User cancelled
          return;
        }

        const writable = await handle.createWritable();
        await writable.write(content);
        await writable.close();

        // Update app state to reflect saved file
        try {
          currentFilePath.set(
            handle.name || (typeof handle === "string" ? handle : null),
          );
        } catch (e) {
          // ignore
        }
        isUnsaved.set(false);
        alert(`Saved to: ${handle.name || "selected file"}`);
        return;
      } catch (err) {
        console.error("SaveFilePicker error:", err);
        // fall through to download fallback
      }
    }

    // If showSaveFilePicker is not available or failed, try showOpenFilePicker to let user pick an existing file to overwrite
    if (win.showOpenFilePicker) {
      try {
        const [handle] = await win.showOpenFilePicker({
          types: [
            {
              description: "Path files",
              accept: { "application/json": [".pp", ".json"] },
            },
          ],
          multiple: false,
        });

        if (handle) {
          const writable = await handle.createWritable();
          await writable.write(content);
          await writable.close();
          try {
            currentFilePath.set(handle.name || null);
          } catch (e) {}
          isUnsaved.set(false);
          alert(`Saved to local file: ${handle.name || "selected file"}`);
          return;
        }
      } catch (err) {
        console.error("showOpenFilePicker error:", err);
        // fall through to download fallback
      }
    }

    // Fallback for browsers without File System Access (e.g., Firefox).
    // Automatically save into the app's browser-backed storage to avoid forcing a download.
    try {
      await saveFile();
      alert(
        "Your browser does not support native file dialogs. The project was saved to the app's storage.\n\nOpen the File Manager to download or export the file to your computer.",
      );
    } catch (err) {
      console.error("Failed to save into app storage:", err);
      // As a last resort, download the file
      try {
        downloadTrajectory(startPoint, lines, shapes, sequence);
      } catch (err2) {
        console.error("Save As fallback failed:", err2);
        alert(
          "Failed to save file. Your browser may not support file picker APIs.",
        );
      }
    }
  }

  function animate(timestamp: number) {
    if (!startTime) {
      startTime = timestamp;
    }

    if (previousTime !== null) {
      const deltaTime = timestamp - previousTime;
      if (percent >= 100) {
        percent = 0;
      } else {
        percent += (0.65 / lines.length) * (deltaTime * 0.1);
      }
    }

    previousTime = timestamp;

    if (playing) {
      requestAnimationFrame(animate);
    }
  }

  function play() {
    animationController.play();
    playing = true;
  }

  function pause() {
    animationController.pause();
    playing = false;
  }

  function resetAnimation() {
    animationController.reset();
    playing = false;
  }

  // Handle slider changes
  function handleSeek(newPercent: number) {
    if (animationController) {
      animationController.seekToPercent(newPercent);
    }
  }

  onMount(() => {
    two = new Two({
      fitted: true,
      type: Two.Types.svg,
    }).appendTo(twoElement);

    updateRobotImageDisplay();

    let currentElem: string | null = null;
    let isDown = false;
    let dragOffset = { x: 0, y: 0 }; // Store offset to prevent snapping to center

    const isLockedPathElem = (id: string | null): boolean => {
      if (!id || !id.startsWith("point")) return false;
      const parts = id.split("-");
      const lineIdx = Number(parts[1]) - 1;
      if (Number.isNaN(lineIdx)) return false;
      if (lineIdx < 0) return false; // startPoint currently not lockable
      return !!lines[lineIdx]?.locked;
    };

    two.renderer.domElement.addEventListener("mousemove", (evt: MouseEvent) => {
      const elem = document.elementFromPoint(evt.clientX, evt.clientY);

      if (isDown && currentElem) {
        const parts = currentElem.split("-");
        const isPathPoint = parts[0] === "point";
        const isShapePoint = parts[0] === "shape";

        // Skip dragging locked paths
        if (isPathPoint) {
          const hitLine = Number(parts[1]) - 1;
          if (hitLine >= 0 && lines[hitLine]?.locked) return;
        }

        // Use simple bounding rect math to match D3 scales which are bound to clientWidth/Height
        const rect = two.renderer.domElement.getBoundingClientRect();
        const xPos = evt.clientX - rect.left;
        const yPos = evt.clientY - rect.top;

        // Get current store values for reactivity
        const currentGridSize = $gridSize;
        const currentSnapToGrid = $snapToGrid;
        const currentShowGrid = $showGrid;

        // Apply drag offset (in inches) to the raw mouse position
        let rawInchX = x.invert(xPos) + dragOffset.x;
        let rawInchY = y.invert(yPos) + dragOffset.y;

        let inchX = rawInchX;
        let inchY = rawInchY;

        // Always apply grid snapping when enabled
        if (currentSnapToGrid && currentShowGrid && currentGridSize > 0) {
          // Force snap to nearest grid point
          inchX = Math.round(rawInchX / currentGridSize) * currentGridSize;
          inchY = Math.round(rawInchY / currentGridSize) * currentGridSize;

          // Clamp to field boundaries
          inchX = Math.max(0, Math.min(FIELD_SIZE, inchX));
          inchY = Math.max(0, Math.min(FIELD_SIZE, inchY));
        }

        // Handle path point dragging
        if (currentElem.startsWith("obstacle-")) {
          // Handle obstacle vertex dragging
          const parts = currentElem.split("-");
          const shapeIdx = Number(parts[1]);
          const vertexIdx = Number(parts[2]);

          shapes[shapeIdx].vertices[vertexIdx].x = inchX;
          shapes[shapeIdx].vertices[vertexIdx].y = inchY;
          shapes = [...shapes];
        } else {
          // Handle path point dragging
          const line = Number(currentElem.split("-")[1]) - 1;
          const point = Number(currentElem.split("-")[2]);

          if (line === -1) {
            // This is the starting point
            if (startPoint.locked) return;
            startPoint.x = inchX;
            startPoint.y = inchY;
          } else if (lines[line]) {
            if (point === 0 && lines[line].endPoint) {
              lines[line].endPoint.x = inchX;
              lines[line].endPoint.y = inchY;
            } else {
              if (lines[line]?.locked) return;
              lines[line].controlPoints[point - 1].x = inchX;
              lines[line].controlPoints[point - 1].y = inchY;
            }
          }
        }
      } else {
        if (
          (elem?.id.startsWith("point") && !isLockedPathElem(elem.id)) ||
          elem?.id.startsWith("obstacle")
        ) {
          two.renderer.domElement.style.cursor = "pointer";
          currentElem = elem.id;
        } else {
          two.renderer.domElement.style.cursor = "auto";
          currentElem = null;
        }
      }
    });

    two.renderer.domElement.addEventListener("mousedown", (evt: MouseEvent) => {
      if (currentElem && isLockedPathElem(currentElem)) {
        isDown = false;
        return;
      }

      isDown = true;

      if (currentElem) {
        const rect = two.renderer.domElement.getBoundingClientRect();
        const mouseX = x.invert(evt.clientX - rect.left);
        const mouseY = y.invert(evt.clientY - rect.top);

        let objectX = 0;
        let objectY = 0;

        if (currentElem.startsWith("obstacle-")) {
          const parts = currentElem.split("-");
          const shapeIdx = Number(parts[1]);
          const vertexIdx = Number(parts[2]);
          if (shapes[shapeIdx]?.vertices[vertexIdx]) {
            objectX = shapes[shapeIdx].vertices[vertexIdx].x;
            objectY = shapes[shapeIdx].vertices[vertexIdx].y;
          }
        } else {
          const line = Number(currentElem.split("-")[1]) - 1;
          const point = Number(currentElem.split("-")[2]);

          if (line === -1) {
            objectX = startPoint.x;
            objectY = startPoint.y;
          } else if (lines[line]) {
            if (point === 0 && lines[line].endPoint) {
              objectX = lines[line].endPoint.x;
              objectY = lines[line].endPoint.y;
            } else if (lines[line].controlPoints[point - 1]) {
              objectX = lines[line].controlPoints[point - 1].x;
              objectY = lines[line].controlPoints[point - 1].y;
            }
          }
        }

        dragOffset = {
          x: objectX - mouseX,
          y: objectY - mouseY,
        };
      }
    });

    two.renderer.domElement.addEventListener("mouseup", () => {
      isDown = false;
      dragOffset = { x: 0, y: 0 };
      recordChange();
    });

    // Double-click on the field to create a new path at that position
    two.renderer.domElement.addEventListener("dblclick", (evt: MouseEvent) => {
      // Ignore dblclicks on existing points/lines
      const elem = document.elementFromPoint(evt.clientX, evt.clientY);
      if (
        elem?.id &&
        (elem.id.startsWith("point") ||
          elem.id.startsWith("obstacle") ||
          elem.id.startsWith("line"))
      ) {
        return;
      }

      const rect = two.renderer.domElement.getBoundingClientRect();
      const rawInchX = x.invert(evt.clientX - rect.left);
      const rawInchY = y.invert(evt.clientY - rect.top);

      // Apply grid snapping if enabled
      const currentGridSize = $gridSize;
      const currentSnapToGrid = $snapToGrid;
      const currentShowGrid = $showGrid;

      let inchX = rawInchX;
      let inchY = rawInchY;

      if (currentSnapToGrid && currentShowGrid && currentGridSize > 0) {
        inchX = Math.round(rawInchX / currentGridSize) * currentGridSize;
        inchY = Math.round(rawInchY / currentGridSize) * currentGridSize;
      }

      // Clamp to field boundaries
      inchX = Math.max(0, Math.min(FIELD_SIZE, inchX));
      inchY = Math.max(0, Math.min(FIELD_SIZE, inchY));

      // Create a new line with endPoint at the clicked position
      const newLine: Line = {
        id: `line-${Math.random().toString(36).slice(2)}`,
        endPoint: {
          x: inchX,
          y: inchY,
          heading: "tangential",
          reverse: false,
        },
        controlPoints: [],
        color: getRandomColor(),
        locked: false,
        waitBeforeMs: 0,
        waitAfterMs: 0,
        waitBeforeName: "",
        waitAfterName: "",
      };

      lines = [...lines, newLine];
      sequence = [...sequence, { kind: "path", lineId: newLine.id! }];
      recordChange();
      two.update();
    });
  });
  document.addEventListener("keydown", function (evt) {
    if (evt.code === "Space" && document.activeElement === document.body) {
      if (playing) {
        pause();
      } else {
        play();
      }
    }
  });
  async function saveFile() {
    try {
      const content = JSON.stringify(
        {
          startPoint,
          lines,
          shapes,
          sequence,
          settings,
          version: "1.2.1",
          timestamp: new Date().toISOString(),
        },
        null,
        2,
      );

      if ($currentFilePath) {
        await browserFileStore.writeFile($currentFilePath, content);
        isUnsaved.set(false);
        // Provide simple feedback
        alert(`Saved to project storage: ${$currentFilePath}`);
      } else {
        // No current project file selected — save into browser cache as a new file
        const defaultName = `path_${Date.now()}.pp`;
        await browserFileStore.writeFile(defaultName, content);
        currentFilePath.set(defaultName);
        isUnsaved.set(false);
        alert(`Saved to project storage as: ${defaultName}`);
      }
    } catch (err) {
      console.error("Failed to save project to storage:", err);
      alert("Failed to save project to browser storage.");
    }
  }

  async function loadFile(evt: Event) {
    const elem = evt.target as HTMLInputElement;
    const file = elem.files?.[0];

    if (!file) return;

    // Check if file is a .pp file
    if (!file.name.endsWith(".pp")) {
      alert("Please select a .pp file");
      // Reset the file input
      elem.value = "";
      return;
    }

    // Parse and load the uploaded file, then cache it into the browser store.
    loadTrajectoryFromFile(evt, async (data) => {
      // Ensure startPoint has all required fields
      startPoint = data.startPoint || {
        x: 72,
        y: 72,
        heading: "tangential",
        reverse: false,
      };

      // Normalize lines with all required fields
      const normalizedLines = normalizeLines(data.lines || []);
      lines = normalizedLines;

      // Derive sequence from data or create default
      sequence = (
        data.sequence && data.sequence.length
          ? data.sequence
          : normalizedLines.map((ln) => ({
              kind: "path",
              lineId: ln.id!,
            }))
      ) as SequenceItem[];

      // Load shapes with defaults
      shapes = data.shapes || [];

      // Load settings (including robot size) if present
      if (data.settings) {
        settings = { ...settings, ...data.settings };
        robotWidth = settings.rWidth;
        robotHeight = settings.rHeight;
      }

      isUnsaved.set(false);
      recordChange();

      // Cache the uploaded file into the browser-backed store for later access
      try {
        const content = JSON.stringify(data);
        await browserFileStore.writeFile(file.name, content);
        currentFilePath.set(file.name);
      } catch (err) {
        console.warn("Failed to cache uploaded file to store:", err);
      }
    });

    // Reset the file input
    elem.value = "";
  }

  // Electron file-copying logic removed — browser store and upload are used instead.

  // Helper function to load data into app state
  function loadData(data: any) {
    // Ensure startPoint has all required fields
    startPoint = data.startPoint || {
      x: 72,
      y: 72,
      heading: "tangential",
      reverse: false,
    };

    // Normalize lines with all required fields
    const normalizedLines = normalizeLines(data.lines || []);
    lines = normalizedLines;

    // Derive sequence from data or create default
    sequence = (
      data.sequence && data.sequence.length
        ? data.sequence
        : normalizedLines.map((ln) => ({
            kind: "path",
            lineId: ln.id!,
          }))
    ) as SequenceItem[];

    // Load shapes with defaults
    shapes = data.shapes || [];

    // Load settings (including robot size) if present
    if (data.settings) {
      settings = { ...settings, ...data.settings };
      robotWidth = settings.rWidth;
      robotHeight = settings.rHeight;
    }

    isUnsaved.set(false);
    recordChange();
  }

  function toHeadingDegrees(point: Point, position: "start" | "end"): number {
    if (!point) return 0;
    if (point.heading === "linear") {
      return position === "start" ? (point.startDeg ?? 0) : (point.endDeg ?? 0);
    }
    if (point.heading === "constant") {
      return point.degrees ?? 0;
    }
    return 0;
  }

  function buildOptimizationPayload(lineIndex: number) {
    const line = lines[lineIndex];
    if (!line) throw new Error("Line not found");

    const startPt =
      lineIndex === 0 ? startPoint : lines[lineIndex - 1]?.endPoint;
    if (!startPt) throw new Error("Missing start point for optimization");

    const waypoints = [startPt, ...line.controlPoints, line.endPoint].map(
      (p) => [p.x, p.y],
    );

    return {
      waypoints,
      start_heading_degrees: toHeadingDegrees(startPt, "start"),
      end_heading_degrees: toHeadingDegrees(line.endPoint, "end"),
      x_velocity: settings.xVelocity,
      y_velocity: settings.yVelocity,
      angular_velocity: settings.aVelocity,
      friction_coefficient: settings.kFriction,
      robot_width: settings.rWidth,
      robot_height: settings.rHeight,
      min_coord_field: 0,
      max_coord_field: FIELD_SIZE,
      interpolation:
        line.endPoint.heading === "tangential"
          ? "tangent"
          : line.endPoint.heading === "constant"
            ? "constant"
            : "linear",
      obstacles: shapes.map((shape) => shape.vertices.map((v) => [v.x, v.y])),
    };
  }

  function sleep(ms: number) {
    return new Promise((res) => setTimeout(res, ms));
  }

  async function runOptimization(payload: any) {
    const response = await fetch(`${OPTIMIZER_BASE_URL}`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(payload),
    });

    if (!response.ok) {
      const errorText = await response.text().catch(() => "");
      throw new Error(
        `Optimizer request failed (${response.status}): ${errorText || response.statusText}`,
      );
    }

    const data = await response.json();
    if (data?.status === "completed" && data.result) {
      return data.result;
    }
    if (data?.status === "error") {
      throw new Error(
        `Optimization failed: ${data.message || "Unknown error"}`,
      );
    }
    throw new Error("Unexpected API response format");
  }

  async function optimizeLine(
    lineId: string,
    targetControlPointIndex?: number,
  ) {
    const lineIndex = lines.findIndex((l) => l.id === lineId);
    if (lineIndex === -1) {
      alert("Could not find line to optimize.");
      return;
    }

    if (optimizingLineIds[lineId]) return;
    optimizingLineIds = { ...optimizingLineIds, [lineId]: true };

    try {
      const payload = buildOptimizationPayload(lineIndex);
      const result = await runOptimization(payload);

      const optimizedWaypoints = Array.isArray(result?.optimized_waypoints)
        ? result.optimized_waypoints
        : Array.isArray(result)
          ? result
          : null;

      if (!optimizedWaypoints || optimizedWaypoints.length < 2) {
        throw new Error("Unexpected optimizer response format.");
      }

      const interior = optimizedWaypoints
        .slice(1, optimizedWaypoints.length - 1)
        .map((p: number[]) => ({ x: p[0], y: p[1] }));

      const newLines = [...lines];
      const current = newLines[lineIndex];

      if (typeof targetControlPointIndex === "number") {
        // Only replace the targeted control point; keep others and endpoint untouched
        const replacement =
          interior[targetControlPointIndex] ?? interior[interior.length - 1];
        if (replacement) {
          const cps = [...current.controlPoints];
          if (cps[targetControlPointIndex]) {
            cps[targetControlPointIndex] = replacement;
            newLines[lineIndex] = {
              ...current,
              controlPoints: cps,
            };
            lines = normalizeLines(newLines);
            recordChange();
          }
        }
      } else {
        // Replace entire line (control points and endpoint)
        newLines[lineIndex] = {
          ...current,
          endPoint: {
            ...current.endPoint,
            x: optimizedWaypoints[optimizedWaypoints.length - 1][0],
            y: optimizedWaypoints[optimizedWaypoints.length - 1][1],
          },
          controlPoints: interior,
        };
        lines = normalizeLines(newLines);
        recordChange();
      }
    } catch (err) {
      console.error(err);
      alert((err as Error).message || "Optimization failed.");
    } finally {
      optimizingLineIds = { ...optimizingLineIds, [lineId]: false };
    }
  }

  async function optimizeAllLines() {
    if (optimizingAll) return;
    optimizingAll = true;
    try {
      for (const ln of lines) {
        if (!ln?.id) continue;
        await optimizeLine(ln.id);
      }
    } finally {
      optimizingAll = false;
    }
  }

  function loadRobot(evt: Event) {
    loadRobotImage(evt, () => updateRobotImageDisplay());
  }

  function addNewLine() {
    lines = [
      ...lines,
      {
        id: `line-${Math.random().toString(36).slice(2)}`,
        endPoint: {
          x: _.random(36, 108),
          y: _.random(36, 108),
          heading: "tangential",
          reverse: true,
        } as Point,
        controlPoints: [],
        color: getRandomColor(),
        locked: false,
        waitBeforeMs: 0,
        waitAfterMs: 0,
        waitBeforeName: "",
        waitAfterName: "",
      },
    ];
    sequence = [
      ...sequence,
      { kind: "path", lineId: lines[lines.length - 1].id! },
    ];
    recordChange();
  }

  function addControlPoint() {
    if (lines.length > 0) {
      const lastLine = lines[lines.length - 1];
      lastLine.controlPoints.push({
        x: _.random(36, 108),
        y: _.random(36, 108),
      });
      recordChange();
    }
  }

  function removeControlPoint() {
    if (lines.length > 0) {
      const lastLine = lines[lines.length - 1];
      if (lastLine.controlPoints.length > 0) {
        lastLine.controlPoints.pop();
        recordChange();
      }
    }
  }

  // Keyboard shortcuts for quick path editing (w is reserved for reset turret below)
  hotkeys("shift+n", function (event, handler) {
    event.preventDefault();
    addNewLine();
  });
  hotkeys("shift+a", function (event, handler) {
    event.preventDefault();
    addControlPoint();
    two.update();
  });
  hotkeys("shift+s", function (event, handler) {
    event.preventDefault();
    removeControlPoint();
    two.update();
  });
  // a/d/s: rotate bot (a = left, d = right, s = reset heading to 90°)
  function ensureManualThenRotate(updateHeading: (current: number) => number) {
    if (manualBotPosition == null) {
      manualBotPosition = clampToField({
        x: x.invert(animationRobotXY.x),
        y: y.invert(animationRobotXY.y),
      });
    }
    const current = manualHeading != null ? manualHeading : animationRobotHeading;
    manualHeading = updateHeading(current);
  }
  hotkeys("a", function (event) {
    event.preventDefault();
    ensureManualThenRotate((h) => h - ROTATION_STEP_DEGREES);
  });
  hotkeys("d", function (event) {
    event.preventDefault();
    ensureManualThenRotate((h) => h + ROTATION_STEP_DEGREES);
  });
  hotkeys("s", function (event) {
    event.preventDefault();
    ensureManualThenRotate(() => -90);
  });
  hotkeys("cmd+z, ctrl+z", function (event) {
    event.preventDefault();
    undoAction();
  });
  hotkeys("cmd+shift+z, ctrl+shift+z, ctrl+y", function (event) {
    event.preventDefault();
    redoAction();
  });

  // Arrow keys: move bot position (field inches). Escape clears manual control.
  function ensureManualThenUpdate(
    update: (pos: { x: number; y: number }) => Partial<{ x: number; y: number }>,
  ) {
    if (manualBotPosition == null) {
      manualBotPosition = clampToField({
        x: x.invert(animationRobotXY.x),
        y: y.invert(animationRobotXY.y),
      });
    }
    manualBotPosition = clampToField({
      ...manualBotPosition,
      ...update(manualBotPosition),
    });
  }
  hotkeys("left", function (event) {
    event.preventDefault();
    ensureManualThenUpdate((p) => ({
      x: Math.max(0, p.x - ARROW_KEY_STEP_INCHES),
    }));
  });
  hotkeys("right", function (event) {
    event.preventDefault();
    ensureManualThenUpdate((p) => ({
      x: Math.min(FIELD_SIZE, p.x + ARROW_KEY_STEP_INCHES),
    }));
  });
  hotkeys("up", function (event) {
    event.preventDefault();
    ensureManualThenUpdate((p) => ({
      y: Math.min(FIELD_SIZE, p.y + ARROW_KEY_STEP_INCHES),
    }));
  });
  hotkeys("down", function (event) {
    event.preventDefault();
    ensureManualThenUpdate((p) => ({
      y: Math.max(0, p.y - ARROW_KEY_STEP_INCHES),
    }));
  });
  hotkeys("escape", function (event) {
    if (manualBotPosition != null || manualHeading != null) {
      manualBotPosition = null;
      manualHeading = null;
      event.preventDefault();
    }
  });
  hotkeys("t", function (event) {
    event.preventDefault();
    useTurretCenterForThetaB = !useTurretCenterForThetaB;
  });
  hotkeys("q", function (event) {
    event.preventDefault();
    turretAngle = Math.max(TURRET_ANGLE_MIN, turretAngle - 1);
  });
  hotkeys("e", function (event) {
    event.preventDefault();
    turretAngle = Math.min(TURRET_ANGLE_MAX, turretAngle + 1);
  });
  hotkeys("w", function (event) {
    event.preventDefault();
    resetAllPositionAndAngles();
  });
  function resetAllPositionAndAngles() {
    // Clear key-driven overrides so next arrow/a/d/s starts from reset position (no accumulated offset)
    manualBotPosition = null;
    manualHeading = null;
    percent = 0; // so timeline-derived animationRobotXY/Heading stay at path start (reactive block overwrites them from percent)
    const startPx = { x: x(startPoint.x), y: y(startPoint.y) };
    const startHeading =
      startPoint.heading === "linear"
        ? -startPoint.startDeg
        : startPoint.heading === "constant"
          ? -startPoint.degrees
          : 0;
    animationRobotXY = startPx;
    animationRobotHeading = startHeading;
    robotXY = startPx;
    robotHeading = startHeading;
    turretAngle = TURRET_ANGLE_CENTER;
  }
  function applyTheme(theme: "light" | "dark" | "auto") {
    let actualTheme = theme;
    if (theme === "auto") {
      // Check system preference
      if (
        window.matchMedia &&
        window.matchMedia("(prefers-color-scheme: dark)").matches
      ) {
        actualTheme = "dark";
      } else {
        actualTheme = "light";
      }
    }

    if (actualTheme === "dark") {
      document.documentElement.classList.add("dark");
    } else {
      document.documentElement.classList.remove("dark");
    }
  }

  // Watch for theme changes in settings
  $: if (settings) {
    applyTheme(settings.theme);
  }

  // Watch for system theme changes if auto mode is enabled
  let mediaQuery: MediaQueryList;
  onMount(() => {
    if (settings?.theme === "auto") {
      mediaQuery = window.matchMedia("(prefers-color-scheme: dark)");
      const handleSystemThemeChange = () => {
        if (settings.theme === "auto") {
          applyTheme("auto");
        }
      };
      mediaQuery.addEventListener("change", handleSystemThemeChange);

      return () => {
        mediaQuery.removeEventListener("change", handleSystemThemeChange);
      };
    }
  });

  // Auto-export for CI/testing: if the app is loaded with URL hash #export-gif-test, automatically run GIF export once mounted
  onMount(() => {
    if (
      typeof window !== "undefined" &&
      window.location &&
      window.location.hash === "#export-gif-test"
    ) {
      // Delay slightly to allow initial rendering and Two.js to initialize
      setTimeout(async () => {
        try {
          // auto GIF export removed (exportGif deleted)
          console.log("Auto GIF export skipped (exportGif removed)");
        } catch (err) {
          console.error("Auto GIF export failed:", err);
        }
      }, 1500);
    }
  });
</script>

<Navbar
  bind:lines
  bind:startPoint
  bind:shapes
  bind:sequence
  bind:settings
  bind:robotWidth
  bind:robotHeight
  bind:useRedGoal
  bind:snapToGoal
  {percent}
  {recordChange}
/>
<!--   {saveFile} -->
<div
  class="w-screen h-screen pt-20 p-2 flex flex-row justify-center items-center gap-2"
>
  <div class="flex h-full justify-center items-center">
    <div
      bind:this={twoElement}
      bind:clientWidth={width}
      bind:clientHeight={height}
      class="h-full aspect-square rounded-lg shadow-md bg-neutral-50 dark:bg-neutral-900 relative overflow-clip"
      role="application"
      style="
    user-select: none;
    -webkit-user-select: none;
    -moz-user-select: none;
    -ms-user-select: none;
    -webkit-touch-callout: none;
    -webkit-tap-highlight-color: transparent;
    user-drag: none;
    -webkit-user-drag: none;
    -khtml-user-drag: none;
    -moz-user-drag: none;
    -ms-user-drag: none;
    -o-user-drag: none;
  "
      on:contextmenu={(e) => e.preventDefault()}
      on:dragstart={(e) => e.preventDefault()}
      on:selectstart={(e) => e.preventDefault()}
      tabindex="-1"
    >
      <img
        src={settings.fieldMap
          ? assetUrl(`fields/${settings.fieldMap}`)
          : assetUrl("fields/decode.webp")}
        alt="Field"
        class="absolute top-0 left-0 w-full h-full rounded-lg z-10"
        style="
    background: transparent; 
    pointer-events: none; 
    user-select: none; 
    -webkit-user-select: none;
    -moz-user-select: none;
    -ms-user-select: none;
    -webkit-touch-callout: none;
    -webkit-tap-highlight-color: transparent;
    user-drag: none;
    -webkit-user-drag: none;
    -moz-user-drag: none;
    -ms-user-drag: none;
    -o-user-drag: none;
  "
        draggable="false"
        on:error={(e) => {
          console.error("Failed to load field map:", settings.fieldMap);
          e.target.src = assetUrl("fields/decode.webp");
        }}
        on:dragstart={(e) => e.preventDefault()}
        on:selectstart={(e) => e.preventDefault()}
      />
      <MathTools {x} {y} {twoElement} {robotXY} {robotHeading} />
      <!-- Robot + turret (turret offset halfway to robot left at heading 90°) -->
      <div
        class="robot-wrapper"
        style="position: absolute; left: {robotXY.x}px; top: {robotXY.y}px; width: {x(robotWidth)}px; height: {x(robotHeight)}px; transform: translate(-50%, -50%) rotate({robotHeading}deg); z-index: 40; pointer-events: none;"
      >
        <img
          src={assetUrl(settings.robotImage || "robot.png")}
          alt="Robot"
          style="
            position: absolute;
            left: 0;
            top: 0;
            width: 100%;
            height: 100%;
            z-index: 0;
            user-select: none;
            -webkit-user-select: none;
            -moz-user-select: none;
            -ms-user-select: none;
          "
          draggable="false"
          on:error={(e) => {
            console.error("Failed to load robot image:", settings.robotImage);
            e.target.src = assetUrl("robot.png");
          }}
          on:dragstart={(e) => e.preventDefault()}
          on:selectstart={(e) => e.preventDefault()}
        />
        <div
          class="turret"
          style="position: absolute; left: 50%; top: 25%; width: {Math.max(20, x(5))}px; height: {Math.max(20, x(5))}px; transform: translate(-50%, -50%); border-radius: 50%; background: #ea580c; border: 3px solid #c2410c; box-sizing: border-box; z-index: 1;"
          aria-hidden="true"
        />
        <!-- INTAKE at 90° (right edge). Full-width bar; rotates with bot (no counter-rotate). -->
        <div
          class="intake-label"
          style="position: absolute; right: -50%; top: 42%; width: 100%; height: fit-content; display: flex; align-items: center; justify-content: center; z-index: 2; padding: 2px 4px; background: #1e40af; color: white; font-size: {Math.max(10, x(1.2))}px; font-weight: 700; font-family: ui-sans-serif, system-ui, sans-serif; white-space: nowrap; border-radius: 2px; border: 1px solid #1e3a8a; box-sizing: border-box; transform: rotate(90deg)"
          aria-hidden="true"
        >
          INTAKE
        </div>
      </div>
    </div>
  </div>
  <ControlTab
    bind:playing
    {play}
    {pause}
    bind:startPoint
    bind:lines
    bind:sequence
    bind:robotWidth
    bind:robotHeight
    bind:settings
    bind:percent
    bind:robotXY
    bind:robotHeading
    bind:shapes
    goalCenter={thetaBTarget}
    bind:useRedGoal
    {thetaBOriginPx}
    {turretCenterPx}
    {turretAngle}
    {resetAllPositionAndAngles}
    {x}
    {y}
    {animationDuration}
    {handleSeek}
    bind:loopAnimation
    {resetAnimation}
    {recordChange}
    {optimizeLine}
    {optimizingLineIds}
    {optimizeAllLines}
    {optimizingAll}
  />
</div>
