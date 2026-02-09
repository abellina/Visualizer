<script lang="ts">
  export let robotXY: BasePoint;
  export let robotHeading: number;
  export let x: d3.ScaleLinear<number, number, number>;
  export let y: d3.ScaleLinear<number, number, number>;
</script>

<div class="flex flex-col w-full justify-start items-start gap-0.5 text-sm rounded-lg border border-neutral-200 dark:border-neutral-700 p-3 bg-white dark:bg-neutral-800/50">
  <div class="font-semibold text-neutral-800 dark:text-neutral-200">Current Robot Position</div>
  <div class="grid grid-cols-[auto_1fr] gap-x-3 gap-y-1 font-mono text-xs">
    <span class="text-neutral-500 dark:text-neutral-400">X</span>
    <span>{x.invert(robotXY.x).toFixed(3)} in</span>
    <span class="text-neutral-500 dark:text-neutral-400">Y</span>
    <span>{y.invert(robotXY.y).toFixed(3)} in</span>
    <span class="text-neutral-500 dark:text-neutral-400">Heading</span>
    <span>{robotHeading.toFixed(0) === "-0" ? "0" : -robotHeading.toFixed(0)}°</span>
  </div>
  <ul class="text-xs text-neutral-500 dark:text-neutral-400 mt-0.5 space-y-0.5 list-none pl-0">
    <li><strong>X, Y</strong> = robot center position in field (inches)</li>
    <li><strong>Heading</strong> = bot forward direction (°)</li>
  </ul>
</div>
