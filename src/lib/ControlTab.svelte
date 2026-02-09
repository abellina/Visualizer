<script lang="ts">
  import type {
    Point,
    Line,
    BasePoint,
    Settings,
    Shape,
    SequenceItem,
  } from "../types";
  import _ from "lodash";
  import { getRandomColor } from "../utils";
  import ObstaclesSection from "./components/ObstaclesSection.svelte";
  import BotToGoalSection from "./components/BotToGoalSection.svelte";
  import StartingPointSection from "./components/StartingPointSection.svelte";
  import PathLineSection from "./components/PathLineSection.svelte";
  import PlaybackControls from "./components/PlaybackControls.svelte";
  import WaitRow from "./components/WaitRow.svelte";
  import { calculatePathTime } from "../utils";

  export let percent: number;
  export let playing: boolean;
  export let play: () => any;
  export let pause: () => any;
  export let startPoint: Point;
  export let lines: Line[];
  export let sequence: SequenceItem[];
  export let robotWidth: number = 16;
  export let robotHeight: number = 16;
  export let robotXY: BasePoint;
  export let robotHeading: number;
  export let x: d3.ScaleLinear<number, number, number>;
  export let y: d3.ScaleLinear<number, number, number>;
  export let settings: Settings;
  export let handleSeek: (percent: number) => void;
  export let loopAnimation: boolean;
  export let optimizeLine: (lineId: string, targetControlPointIndex?: number) => void;
  export let optimizingLineIds: Record<string, boolean> = {};

  export let shapes: Shape[];
  export let goalCenter: { x: number; y: number };
  /** When true, red goal; when false, blue goal. Toggle with button. */
  export let useRedGoal: boolean = true;
  /** Origin for θ_b (bot or turret center in pixels). Used by BotToGoalSection for x_b, y_b, θ_b. */
  export let thetaBOriginPx: { x: number; y: number };
  /** Turret center in pixels (for "turret angle to face goal" using actual turret position). */
  export let turretCenterPx: { x: number; y: number };
  export let recordChange: () => void;

  const TURRET_CENTER_DEG = 95;
  const TURRET_MIN = 0;
  const TURRET_MAX = 190;
  /** Normalize desired turret angle to [0, 190]. */
  function clampTurret(deg: number): number {
    return Math.max(TURRET_MIN, Math.min(TURRET_MAX, deg));
  }
  /** Wrap angle difference to (-180, 180] for shortest turn. */
  function wrapAngleDeg(deg: number): number {
    let d = deg % 360;
    if (d > 180) d -= 360;
    if (d <= -180) d += 360;
    return d;
  }

  // Reference exported but unused props to silence Svelte unused-export warnings

  $: robotWidth;
  $: robotHeight;

  // Compute timeline markers for the UI (start of each travel segment)
  $: timePrediction = calculatePathTime(startPoint, lines, settings, sequence);
  $: markers = (() => {
    const _markers: { percent: number; color: string; name: string }[] = [];
    if (
      !timePrediction ||
      !timePrediction.timeline ||
      timePrediction.totalTime <= 0
    )
      return _markers;

    timePrediction.timeline.forEach((ev) => {
      if ((ev as any).type === "travel") {
        const end = (ev as any).endTime as number;
        const pct = (end / timePrediction.totalTime) * 100;
        const lineIndex = (ev as any).lineIndex as number;
        const line = lines[lineIndex];
        const color = line?.color || "#ffffff";
        const name = line?.name || `Path ${lineIndex + 1}`;
        _markers.push({ percent: pct, color, name });
      }
    });

    return _markers;
  })();


  /** Turret angle (0–190°, center 95) for display. */
  export let turretAngle: number = 95;
  /** Reset bot position to path start, heading from start point, and turret to 95°. */
  export let resetAllPositionAndAngles: (() => void) | undefined = undefined;

  // θ_b: angle from origin (bot or turret) to goal
  $: theta_b_deg = (() => {
    const xb = x.invert(thetaBOriginPx.x);
    const yb = y.invert(thetaBOriginPx.y);
    const xg = goalCenter.x;
    const yg = goalCenter.y;
    const rad = Math.atan2(yg - yb, xg - xb);
    return (rad * 180) / Math.PI;
  })();

  // Turret angle to face goal: two computation options
  $: theta_b_from_bot_deg = (() => {
    const xb = x.invert(robotXY.x);
    const yb = y.invert(robotXY.y);
    const rad = Math.atan2(goalCenter.y - yb, goalCenter.x - xb);
    return (rad * 180) / Math.PI;
  })();
  $: theta_b_from_turret_deg = (() => {
    const xt = x.invert(turretCenterPx.x);
    const yt = y.invert(turretCenterPx.y);
    const rad = Math.atan2(goalCenter.y - yt, goalCenter.x - xt);
    return (rad * 180) / Math.PI;
  })();
  // Turret direction in field = −robotHeading − (θ_t − 95). For turret to face goal: θ_b = −robotHeading − (θ_t − 95) → θ_t = 95 − robotHeading − θ_b.
  // So desired θ_t = 95 − (θ_b + robotHeading), wrapped so we pick shortest turn then clamp [0, 190].
  function desiredTurretFromAngleToGoal(angleToGoalDeg: number): number {
    const delta = wrapAngleDeg(angleToGoalDeg + robotHeading); // θ_b + θ_h (goal relative to forward)
    return clampTurret(TURRET_CENTER_DEG - delta); // 95 − (θ_b + robotHeading) matches drawing
  }
  // Depend on robotHeading so desired turret updates when bot rotates (θ_b in field is unchanged but angle-from-forward changes).
  $: desired_turret_at_bot_center = (robotHeading, desiredTurretFromAngleToGoal(theta_b_from_bot_deg));
  $: desired_turret_at_turret_pos = (robotHeading, desiredTurretFromAngleToGoal(theta_b_from_turret_deg));
  $: turn_to_goal_bot_center = wrapAngleDeg(desired_turret_at_bot_center - turretAngle);
  $: turn_to_goal_turret_pos = wrapAngleDeg(desired_turret_at_turret_pos - turretAngle);

  // State for collapsed sections (kept for potential future use)
  let collapsedSections = {
    obstacles: shapes.map(() => true),
    lines: lines.map(() => false),
    controlPoints: lines.map(() => true), // Start with control points collapsed
  };

  // Collapsed state for obstacles (default collapsed)
  let collapsedObstacles = shapes.map(() => true);

  // Reactive statements to update UI state when lines or shapes change from file load
  $: if (lines.length !== collapsedSections.lines.length) {
    collapsedSections = {
      obstacles: collapsedSections.obstacles ?? shapes.map(() => true),
      lines: lines.map(() => false),
      controlPoints: lines.map(() => true),
    };
  }

  // Keep obstacle collapse state aligned with shapes list
  $: if (shapes.length !== collapsedObstacles.length) {
    collapsedObstacles = shapes.map(() => true);
  }

  $: if (!collapsedSections.obstacles || shapes.length !== collapsedSections.obstacles.length) {
    collapsedSections = {
      ...collapsedSections,
      obstacles: shapes.map(() => true),
    };
  }

  const makeId = () =>
    `${Date.now().toString(36)}-${Math.random().toString(36).slice(2, 8)}`;
  function getWait(i: any) {
    return i as any;
  }

  function insertLineAfter(seqIndex: number) {
    const seqItem = sequence[seqIndex];
    if (!seqItem || seqItem.kind !== "path") return;
    const lineIndex = lines.findIndex((l) => l.id === seqItem.lineId);
    const currentLine = lines[lineIndex];

    // Find the next path item in the sequence after seqIndex
    let nextPathSeqIndex = -1;
    for (let i = seqIndex + 1; i < sequence.length; i++) {
      if (sequence[i].kind === "path") {
        nextPathSeqIndex = i;
        break;
      }
    }

    // If there is no next path in sequence, fall back to addLine behavior (append new randomized point)
    let newPoint: Point | null = null;
    if (nextPathSeqIndex !== -1) {
      const nextLineId = (sequence[nextPathSeqIndex] as any).lineId;
      const nextLine = lines.find((l) => l.id === nextLineId);
      if (
        nextLine &&
        nextLine.endPoint &&
        currentLine &&
        currentLine.endPoint
      ) {
        const a = currentLine.endPoint;
        const b = nextLine.endPoint;
        const midX = (Number(a.x) + Number(b.x)) / 2;
        const midY = (Number(a.y) + Number(b.y)) / 2;
        newPoint = {
          x: midX,
          y: midY,
          heading: "tangential",
          reverse: false,
        };
      }
    }

    if (!newPoint) {
      // fallback: random nearby point from current end
      if (currentLine && currentLine.endPoint) {
        newPoint = {
          x: (currentLine.endPoint.x ?? 72) + _.random(-12, 12),
          y: (currentLine.endPoint.y ?? 72) + _.random(-12, 12),
          heading: "tangential",
          reverse: false,
        };
      } else {
        newPoint = {
          x: _.random(0, 144),
          y: _.random(0, 144),
          heading: "tangential",
          reverse: false,
        };
      }
    }

    const newLine = {
      id: makeId(),
      endPoint: newPoint,
      controlPoints: [],
      color: getRandomColor(),
      name: `Path ${lines.length + 1}`,
      waitBeforeMs: 0,
      waitAfterMs: 0,
      waitBeforeName: "",
      waitAfterName: "",
    };

    // Insert the new line after the current one and a sequence item after current seq index
    const newLines = [...lines];
    newLines.splice(lineIndex + 1, 0, newLine);
    lines = newLines;

    const newSeq = [...sequence];
    newSeq.splice(seqIndex + 1, 0, { kind: "path", lineId: newLine.id! });
    sequence = newSeq;

    collapsedSections.lines.splice(lineIndex + 1, 0, false);
    collapsedSections.controlPoints.splice(lineIndex + 1, 0, true);

    // Force reactivity
    collapsedSections = { ...collapsedSections };
  }

  // Insert a midpoint between this path and the next path in sequence
  function insertMidpointAfter(seqIndex: number) {
    const seqItem = sequence[seqIndex];
    if (!seqItem || seqItem.kind !== "path") return;
    const lineIndex = lines.findIndex((l) => l.id === seqItem.lineId);
    const currentLine = lines[lineIndex];

    // Find the next path in sequence
    let nextPathSeqIndex = -1;
    for (let i = seqIndex + 1; i < sequence.length; i++) {
      if (sequence[i].kind === "path") {
        nextPathSeqIndex = i;
        break;
      }
    }

    if (nextPathSeqIndex === -1) {
      // no next path -> do nothing or fallback
      return;
    }

    const nextLineId = (sequence[nextPathSeqIndex] as any).lineId;
    const nextLine = lines.find((l) => l.id === nextLineId);
    if (!currentLine || !nextLine) return;

    const a = currentLine.endPoint;
    const b = nextLine.endPoint;
    const midX = (Number(a.x) + Number(b.x)) / 2;
    const midY = (Number(a.y) + Number(b.y)) / 2;

    const newLine: Line = {
      id: makeId(),
      endPoint: {
        x: midX,
        y: midY,
        heading: "tangential",
        reverse: false,
      },
      controlPoints: [],
      color: getRandomColor(),
      name: `Path ${lines.length + 1}`,
      waitBeforeMs: 0,
      waitAfterMs: 0,
      waitBeforeName: "",
      waitAfterName: "",
    };

    // Insert into lines right after current line index
    const newLines = [...lines];
    newLines.splice(lineIndex + 1, 0, newLine);
    lines = newLines;

    // Insert into sequence right after seqIndex
    const newSeq = [...sequence];
    newSeq.splice(seqIndex + 1, 0, { kind: "path", lineId: newLine.id! });
    sequence = newSeq;

    collapsedSections.lines.splice(lineIndex + 1, 0, false);
    collapsedSections.controlPoints.splice(lineIndex + 1, 0, true);

    collapsedSections = { ...collapsedSections };
    recordChange();
  }

  function removeLine(idx: number) {
    const removedId = lines[idx]?.id;
    let _lns = lines;
    lines.splice(idx, 1);
    lines = _lns;
    if (removedId) {
      sequence = sequence.filter(
        (s) => s.kind === "wait" || s.lineId !== removedId,
      );
    }
    collapsedSections.lines.splice(idx, 1);
    collapsedSections.controlPoints.splice(idx, 1);
    recordChange();
  }

  function addLine() {
    const newLine: Line = {
      id: makeId(),
      name: `Path ${lines.length + 1}`,
      endPoint: {
        x: _.random(0, 144),
        y: _.random(0, 144),
        heading: "tangential",
        reverse: false,
      },
      controlPoints: [],
      color: getRandomColor(),
      waitBeforeMs: 0,
      waitAfterMs: 0,
      waitBeforeName: "",
      waitAfterName: "",
    };
    lines = [...lines, newLine];
    sequence = [...sequence, { kind: "path", lineId: newLine.id! }];
    collapsedSections.lines.push(false);
    collapsedSections.controlPoints.push(true);
    recordChange();
  }

  // Add a control point to the line represented by `seqIndex` in the sequence
  function addControlPointToLine(seqIndex: number) {
    const seqItem = sequence[seqIndex];
    if (!seqItem || seqItem.kind !== "path") return;
    const lineIndex = lines.findIndex((l) => l.id === seqItem.lineId);
    if (lineIndex === -1) return;
    const line = lines[lineIndex];
    line.controlPoints = line.controlPoints || [];
    const prevPt = lineIndex === 0 ? startPoint : lines[lineIndex - 1].endPoint;
    const endPt = line.endPoint || { x: 72, y: 72 };
    const mx = ((prevPt?.x ?? 72) + (endPt?.x ?? 72)) / 2;
    const my = ((prevPt?.y ?? 72) + (endPt?.y ?? 72)) / 2;
    line.controlPoints.push({
      x: mx + _.random(-4, 4),
      y: my + _.random(-4, 4),
    });
    collapsedSections.controlPoints[lineIndex] = false;
    lines = [...lines];
    collapsedSections = { ...collapsedSections };
    recordChange?.();
  }

  // Add a control point to the last path in `lines` (fallback: create a new line)
  function addControlPointToLastLine() {
    if (!lines || lines.length === 0) {
      // No lines exist: create a new line instead
      addLine();
      return;
    }

    // Prefer adding to the first line whose control points are expanded (user is focusing it)
    let targetIdx = collapsedSections.controlPoints.findIndex(
      (v) => v === false,
    );
    if (targetIdx === -1) targetIdx = lines.length - 1;

    const line = lines[targetIdx];
    line.controlPoints = line.controlPoints || [];
    // Insert a control point near the line midpoint for convenience
    const prevPt = targetIdx === 0 ? startPoint : lines[targetIdx - 1].endPoint;
    const endPt = line.endPoint || { x: 72, y: 72 };
    const mx = ((prevPt?.x ?? 72) + (endPt?.x ?? 72)) / 2;
    const my = ((prevPt?.y ?? 72) + (endPt?.y ?? 72)) / 2;
    line.controlPoints.push({
      x: mx + _.random(-4, 4),
      y: my + _.random(-4, 4),
    });
    // Ensure control points UI is expanded for this line
    collapsedSections.controlPoints[targetIdx] = false;
    lines = [...lines];
    collapsedSections = { ...collapsedSections };
    recordChange?.();
  }

  function addWait() {
    const wait = {
      kind: "wait",
      id: makeId(),
      name: "Wait",
      durationMs: 0,
      locked: false,
    } as SequenceItem;
    sequence = [...sequence, wait];
  }

  function addWaitAtStart() {
    const wait = {
      kind: "wait",
      id: makeId(),
      name: "Wait",
      durationMs: 0,
      locked: false,
    } as SequenceItem;
    sequence = [wait, ...sequence];
  }

  function addPathAtStart() {
    const newLine: Line = {
      id: makeId(),
      name: `Path ${lines.length + 1}`,
      endPoint: {
        x: _.random(0, 144),
        y: _.random(0, 144),
        heading: "tangential",
        reverse: false,
      },
      controlPoints: [],
      color: getRandomColor(),
      waitBeforeMs: 0,
      waitAfterMs: 0,
      waitBeforeName: "",
      waitAfterName: "",
    };
    lines = [newLine, ...lines];
    sequence = [{ kind: "path", lineId: newLine.id! }, ...sequence];
    collapsedSections.lines = [false, ...collapsedSections.lines];
    collapsedSections.controlPoints = [
      true,
      ...collapsedSections.controlPoints,
    ];
    recordChange();
  }

  function insertWaitAfter(seqIndex: number) {
    const newSeq = [...sequence];
    newSeq.splice(seqIndex + 1, 0, {
      kind: "wait",
      id: makeId(),
      name: "Wait",
      durationMs: 0,
      locked: false,
    });
    sequence = newSeq;
  }

  function insertPathAfter(seqIndex: number) {
    // Create a new line with default settings
    const newLine: Line = {
      id: makeId(),
      name: `Path ${lines.length + 1}`,
      endPoint: {
        x: _.random(36, 108),
        y: _.random(36, 108),
        heading: "tangential",
        reverse: false,
      },
      controlPoints: [],
      color: getRandomColor(),
      waitBeforeMs: 0,
      waitAfterMs: 0,
      waitBeforeName: "",
      waitAfterName: "",
    };

    // Add the new line to the lines array
    lines = [...lines, newLine];

    // Insert the new path in the sequence after the wait
    const newSeq = [...sequence];
    newSeq.splice(seqIndex + 1, 0, { kind: "path", lineId: newLine.id! });
    sequence = newSeq;

    // Add UI state for the new line
    collapsedSections.lines.push(false);
    collapsedSections.controlPoints.push(true);

    // Force reactivity
    collapsedSections = { ...collapsedSections };
    recordChange();
  }

  function syncLinesToSequence(newSeq: SequenceItem[]) {
    const pathOrder = newSeq
      .filter((item) => item.kind === "path")
      .map((item) => item.lineId);

    const indexedLines = lines.map((line, idx) => ({
      line,
      collapsed: collapsedSections.lines[idx],
      control: collapsedSections.controlPoints[idx],
    }));

    const byId = new Map(indexedLines.map((entry) => [entry.line.id, entry]));
    const reordered: typeof indexedLines = [];

    pathOrder.forEach((id) => {
      const entry = byId.get(id);
      if (entry) {
        reordered.push(entry);
        byId.delete(id);
      }
    });

    // Append any lines that are not currently in the sequence to preserve data
    reordered.push(...byId.values());

    lines = reordered.map((entry) => entry.line);
    collapsedSections = {
      ...collapsedSections,
      lines: reordered.map((entry) => entry.collapsed ?? false),
      controlPoints: reordered.map((entry) => entry.control ?? true),
    };
    // No collapsedEventMarkers to update
  }

  function moveSequenceItem(seqIndex: number, delta: number) {
    const targetIndex = seqIndex + delta;
    if (targetIndex < 0 || targetIndex >= sequence.length) return;

    // Prevent moving if either the source or target is a locked path or a locked wait
    const isLockedSequenceItem = (index: number) => {
      const it = sequence[index];
      if (!it) return false;
      if (it.kind === "path") {
        const ln = lines.find((l) => l.id === it.lineId);
        return ln?.locked ?? false;
      }
      // wait
      if (it.kind === "wait") {
        return (it as any).locked ?? false;
      }
      return false;
    };

    if (isLockedSequenceItem(seqIndex) || isLockedSequenceItem(targetIndex))
      return;

    const newSeq = [...sequence];
    const [item] = newSeq.splice(seqIndex, 1);
    newSeq.splice(targetIndex, 0, item);
    sequence = newSeq;

    syncLinesToSequence(newSeq);
    recordChange?.();
  }
</script>

<div class="flex-1 flex flex-col justify-start items-center gap-2 h-full">
  <div
    class="flex flex-col justify-start items-start w-full rounded-lg bg-neutral-50 dark:bg-neutral-900 shadow-md p-4 overflow-y-scroll overflow-x-hidden flex-1 min-h-0 gap-6 border border-neutral-200 dark:border-neutral-700"
  >
    <BotToGoalSection
      originXY={thetaBOriginPx}
      {x}
      {y}
      goal={goalCenter}
      goalLabel={useRedGoal ? "Red Alliance Red Goal" : "Blue Alliance Blue Goal"}
    />

    <!-- Angles: θₕ, θₜ, θᵦ with definitions -->
    <div class="flex flex-col w-full gap-2 text-sm rounded-lg border border-neutral-200 dark:border-neutral-700 p-3 bg-white dark:bg-neutral-800/50">
      <div class="font-semibold text-neutral-800 dark:text-neutral-200">Angles</div>
      <div class="grid grid-cols-[auto_1fr] gap-x-3 gap-y-1 font-mono text-xs">
        <span class="text-neutral-500 dark:text-neutral-400">θ<sub>h</sub></span>
        <span>{(-robotHeading).toFixed(1)}°</span>
        <span class="text-neutral-500 dark:text-neutral-400">θ<sub>t</sub></span>
        <span>{turretAngle.toFixed(1)}°</span>
        <span class="text-neutral-500 dark:text-neutral-400">θ<sub>b</sub></span>
        <span>{theta_b_deg.toFixed(2)}°</span>
      </div>
      <ul class="text-xs text-neutral-500 dark:text-neutral-400 mt-1 space-y-0.5 list-none pl-0">
        <li><strong>θ<sub>h</sub></strong> = heading (bot forward)</li>
        <li><strong>θ<sub>t</sub></strong> = turret angle in bot frame (0–190°, 95 = forward)</li>
        <li><strong>θ<sub>b</sub></strong> = angle from origin to goal</li>
        <li>Press <kbd class="px-0.5 rounded bg-neutral-200 dark:bg-neutral-600 font-mono">T</kbd> to use turret center for θ<sub>b</sub></li>
      </ul>
    </div>

    <!-- Turret angle to face goal: both computation methods with full detail -->
    <div class="flex flex-col w-full gap-3 text-sm rounded-lg border border-neutral-200 dark:border-neutral-700 p-3 bg-white dark:bg-neutral-800/50">
      <div class="font-semibold text-neutral-800 dark:text-neutral-200">
        Turret angle to face goal
      </div>
      <p class="text-xs text-neutral-500 dark:text-neutral-400">
        Turret field direction = −θ<sub>h</sub> − (θ<sub>t</sub> − 95). Set equal to θ<sub>b</sub>: desired θ<sub>t</sub> = 95 − (θ<sub>b</sub> − θ<sub>h</sub>) (wrapped to shortest turn), then clamp to [0, 190].
      </p>
      <div class="text-xs font-mono text-neutral-600 dark:text-neutral-400">
        <span class="text-neutral-500">Turret angle now (θ<sub>t</sub>):</span> {turretAngle.toFixed(1)}° — same for both methods below. Turn by = desired θ<sub>t</sub> − current; + → <kbd class="px-0.5 rounded bg-neutral-200 dark:bg-neutral-600">e</kbd>, − → <kbd class="px-0.5 rounded bg-neutral-200 dark:bg-neutral-600">q</kbd>.
      </div>

      <div class="flex flex-col gap-2 text-xs">
        <div class="rounded border border-neutral-200 dark:border-neutral-600 p-2 space-y-1">
          <div class="font-medium text-neutral-700 dark:text-neutral-300">1) Assume turret at bot center (simplest)</div>
          <div class="text-neutral-600 dark:text-neutral-400">
            Angle from bot to goal (field): θ<sub>b,bot</sub> = atan2(y<sub>g</sub> − y<sub>b</sub>, x<sub>g</sub> − x<sub>b</sub>) = <span class="font-mono">{theta_b_from_bot_deg.toFixed(2)}°</span>
          </div>
          <div class="text-neutral-600 dark:text-neutral-400">
            θ<sub>b,bot</sub> − θ<sub>h</sub> = {theta_b_from_bot_deg.toFixed(2)} − ({(-robotHeading).toFixed(1)}) = <span class="font-mono">{(theta_b_from_bot_deg - (-robotHeading)).toFixed(1)}°</span>. Desired θ<sub>t</sub> = 95 − (that) → <span class="font-mono font-semibold">{desired_turret_at_bot_center.toFixed(1)}°</span>. Turn by: <span class="font-mono">{turn_to_goal_bot_center >= 0 ? "+" : ""}{turn_to_goal_bot_center.toFixed(1)}°</span>
          </div>
        </div>

        <div class="rounded border border-neutral-200 dark:border-neutral-600 p-2 space-y-1">
          <div class="font-medium text-neutral-700 dark:text-neutral-300">2) Use actual turret position</div>
          <div class="text-neutral-600 dark:text-neutral-400">
            Angle from turret to goal (field): θ<sub>b,turret</sub> = atan2(y<sub>g</sub> − y<sub>t</sub>, x<sub>g</sub> − x<sub>t</sub>) = <span class="font-mono">{theta_b_from_turret_deg.toFixed(2)}°</span>
          </div>
          <div class="text-neutral-600 dark:text-neutral-400">
            θ<sub>b,turret</sub> − θ<sub>h</sub> = {theta_b_from_turret_deg.toFixed(2)} − ({(-robotHeading).toFixed(1)}) = <span class="font-mono">{(theta_b_from_turret_deg - (-robotHeading)).toFixed(1)}°</span>. Desired θ<sub>t</sub> = 95 − (that) → <span class="font-mono font-semibold">{desired_turret_at_turret_pos.toFixed(1)}°</span>. Turn by: <span class="font-mono">{turn_to_goal_turret_pos >= 0 ? "+" : ""}{turn_to_goal_turret_pos.toFixed(1)}°</span>
          </div>
        </div>
      </div>
    </div>

    <div class="flex flex-col w-full gap-2 text-sm rounded-lg border border-neutral-200 dark:border-neutral-700 p-3 bg-white dark:bg-neutral-800/50">
      <div class="font-semibold text-neutral-800 dark:text-neutral-200">
        Move bot & turret
      </div>
      {#if resetAllPositionAndAngles}
        <button
          type="button"
          class="px-3 py-1.5 rounded bg-neutral-200 dark:bg-neutral-600 hover:bg-neutral-300 dark:hover:bg-neutral-500 text-neutral-800 dark:text-neutral-200 text-xs font-medium"
          on:click={resetAllPositionAndAngles}
        >
          Reset position & angles
        </button>
      {/if}
      <ul class="list-disc list-inside text-xs text-neutral-600 dark:text-neutral-400 space-y-1">
        <li><kbd class="px-1 py-0.5 rounded bg-neutral-200 dark:bg-neutral-600 font-mono">↑↓←→</kbd> Move bot (2" step)</li>
        <li><kbd class="px-1 py-0.5 rounded bg-neutral-200 dark:bg-neutral-600 font-mono">a</kbd> / <kbd class="px-1 py-0.5 rounded bg-neutral-200 dark:bg-neutral-600 font-mono">d</kbd> Rotate bot left / right (1°)</li>
        <li><kbd class="px-1 py-0.5 rounded bg-neutral-200 dark:bg-neutral-600 font-mono">s</kbd> Reset bot heading to 90°</li>
        <li><kbd class="px-1 py-0.5 rounded bg-neutral-200 dark:bg-neutral-600 font-mono">q</kbd> / <kbd class="px-1 py-0.5 rounded bg-neutral-200 dark:bg-neutral-600 font-mono">e</kbd> Rotate turret left / right (1°, 0–190°)</li>
        <li><kbd class="px-1 py-0.5 rounded bg-neutral-200 dark:bg-neutral-600 font-mono">w</kbd> Reset position, heading & turret to path start</li>
        <li><kbd class="px-1 py-0.5 rounded bg-neutral-200 dark:bg-neutral-600 font-mono">t</kbd> Use turret center for θ<sub>b</sub> (toggle)</li>
        <li><kbd class="px-1 py-0.5 rounded bg-neutral-200 dark:bg-neutral-600 font-mono">Escape</kbd> Clear manual position</li>
      </ul>
    </div>
  </div>
</div>
