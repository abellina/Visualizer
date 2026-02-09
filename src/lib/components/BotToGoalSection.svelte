<script lang="ts">
  import type * as d3 from "d3";
  import type { BasePoint } from "../../types";

  /** Origin for θ_b (bot or turret center in pixels). */
  export let originXY: BasePoint;
  export let x: d3.ScaleLinear<number, number, number>;
  export let y: d3.ScaleLinear<number, number, number>;
  export let goal: { x: number; y: number };
  export let goalLabel: string = "Red Alliance Red Goal";

  // Origin position in field coordinates (inches) for x_b, y_b, θ_b
  $: xb = x.invert(originXY.x);
  $: yb = y.invert(originXY.y);
  $: xg = goal.x;
  $: yg = goal.y;

</script>

<div class="flex flex-col w-full justify-start items-start gap-2 text-sm rounded-lg border border-neutral-200 dark:border-neutral-700 p-3 bg-white dark:bg-neutral-800/50">
  <div class="font-semibold text-neutral-800 dark:text-neutral-200">
    Bot to Goal — {goalLabel}
  </div>
  <div class="font-mono text-xs text-neutral-700 dark:text-neutral-300 flex flex-col gap-0.5">
    <div>(x<sub>b</sub>, y<sub>b</sub>) = ({xb.toFixed(3)}, {yb.toFixed(3)}) in</div>
    <div>(x<sub>g</sub>, y<sub>g</sub>) = ({xg.toFixed(3)}, {yg.toFixed(3)}) in</div>
  </div>
  <ul class="text-xs text-neutral-500 dark:text-neutral-400 mt-0.5 space-y-0.5 list-none pl-0">
    <li><strong>x<sub>b</sub>, y<sub>b</sub></strong> = origin position in field (in) — bot or turret when <kbd class="px-0.5 rounded bg-neutral-200 dark:bg-neutral-600 font-mono">T</kbd> toggled</li>
    <li><strong>x<sub>g</sub>, y<sub>g</sub></strong> = goal position in field (in)</li>
  </ul>
</div>
