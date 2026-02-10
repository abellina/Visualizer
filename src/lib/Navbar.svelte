<script lang="ts">
  import type { Point, Line, Shape, Settings, SequenceItem } from "../types";
  import { onMount } from "svelte";
  import {
    showRuler,
    showProtractor,
    showGrid,
    protractorLockToRobot,
    gridSize,
    currentFilePath,
    isUnsaved,
    snapToGrid,
  } from "../stores";
  import { getRandomColor } from "../utils";
  import {
    getDefaultStartPoint,
    getDefaultLines,
    getDefaultShapes,
  } from "../config";
  import FileManager from "./FileManager.svelte";
  import SettingsDialog from "./components/SettingsDialog.svelte";
  export let startPoint: Point;
  export let lines: Line[];
  export let shapes: Shape[];
  export let sequence: SequenceItem[];
  export let percent: number = 0;
  export let robotWidth: number;
  export let robotHeight: number;
  export let settings: Settings;

  export let recordChange: () => any;
  /** When true, red goal; when false, blue goal. Toggle button in bar. */
  export let useRedGoal: boolean = true;
  /** When false: snap to tag (θ_b to goal center). When true: snap to goal (θ_b extended 6" in +x). */
  export let snapToGoal: boolean = false;

  let fileManagerOpen = false;
  let settingsOpen = false;

  let selectedGridSize = 12;
  const gridSizeOptions = [1, 3, 6, 12, 24];

  onMount(() => {
    const unsubscribeGridSize = gridSize.subscribe((value) => {
      selectedGridSize = value;
    });

    return () => {
      unsubscribeGridSize();
    };
  });

  function cycleGridSize() {
    if (!$showGrid) {
      // Grid is off, turn it on with first size
      showGrid.set(true);
      selectedGridSize = gridSizeOptions[0];
      gridSize.set(selectedGridSize);
    } else {
      // Grid is on, cycle to next size or turn off
      const currentIndex = gridSizeOptions.indexOf(selectedGridSize);
      const nextIndex = currentIndex + 1;
      if (nextIndex >= gridSizeOptions.length) {
        // We're at the last size, turn off
        showGrid.set(false);
      } else {
        // Move to next size
        selectedGridSize = gridSizeOptions[nextIndex];
        gridSize.set(selectedGridSize);
      }
    }
  }

  function resetPath() {
    startPoint = getDefaultStartPoint();
    lines = getDefaultLines();
    sequence = lines.map((ln) => ({
      kind: "path",
      lineId: ln.id || `line-${Math.random().toString(36).slice(2)}`,
    }));
    shapes = getDefaultShapes();
  }

  function handleResetPathWithConfirmation() {
    // Check if there's unsaved work
    const hasChanges = $isUnsaved || lines.length > 1 || shapes.length > 0;

    let message = "Are you sure you want to reset the path?\n\n";

    if (hasChanges) {
      if ($currentFilePath) {
        message += `This will reset "${$currentFilePath.split(/[\\/]/).pop()}" to the default path.`;
      } else {
        message += "This will reset your current work to the default path.";
      }

      if ($isUnsaved) {
        message += "\n\n⚠ WARNING: You have unsaved changes that will be lost!";
      }
    } else {
      message += "This will reset to the default starting path.";
    }

    message += "\n\nClick OK to reset, or Cancel to keep your current path.";

    if (confirm(message)) {
      resetPath();
      if (recordChange) recordChange();
    }
  }

  $: if (settings) {
    settings.rHeight = robotHeight;
    settings.rWidth = robotWidth;
  }

</script>

{#if fileManagerOpen}
  <FileManager
    bind:isOpen={fileManagerOpen}
    bind:startPoint
    bind:lines
    bind:shapes
    bind:sequence
  />
{/if}

<SettingsDialog bind:isOpen={settingsOpen} bind:settings />

<div
  class="absolute top-0 left-0 w-full bg-neutral-50 dark:bg-neutral-900 shadow-md flex flex-row justify-between items-center px-6 py-4 border-b-[0.75px] border-[#b300e6]"
>
  <!-- Title -->
  <div class="font-semibold flex flex-col justify-start items-start">
    <div class="flex flex-row items-center gap-2">
      <!-- File manager button -->
      <button
        title="File Manager"
        on:click={() => (fileManagerOpen = true)}
      >
        <svg
          xmlns="http://www.w3.org/2000/svg"
          fill="none"
          viewBox="0 0 24 24"
          stroke-width="2"
          stroke="currentColor"
          class="size-6"
        >
          <path
            stroke-linecap="round"
            stroke-linejoin="round"
            d="M3.75 6.75h16.5M3.75 12h16.5m-16.5 5.25h16.5"
          />
        </svg>
      </button>

      <span>Ctrl-X Angle Visualizer (based on Pedro Pathing Visualizer)</span>
      <!-- GitHub Repo Link (moved next to title) -->
      <a
        target="_blank"
        rel="noreferrer"
        title="GitHub Repo"
        href="https://github.com/Pedro-Pathing/Visualizer"
      >
        <svg
          xmlns="http://www.w3.org/2000/svg"
          viewBox="0 0 30 30"
          class="size-6 dark:fill-white"
        >
          <path
            d="M15,3C8.373,3,3,8.373,3,15c0,5.623,3.872,10.328,9.092,11.63C12.036,26.468,12,26.28,12,26.047v-2.051 c-0.487,0-1.303,0-1.508,0c-0.821,0-1.551-0.353-1.905-1.009c-0.393-0.729-0.461-1.844-1.435-2.526 c-0.289-0.227-0.069-0.486,0.264-0.451c0.615,0.174,1.125,0.596,1.605,1.222c0.478,0.627,0.703,0.769,1.596,0.769 c0.433,0,1.081-0.025,1.691-0.121c0.328-0.833,0.895-1.6,1.588-1.962c-3.996-0.411-5.903-2.399-5.903-5.098 c0-1.162,0.495-2.286,1.336-3.233C9.053,10.647,8.706,8.73,9.435,8c1.798,0,2.885,1.166,3.146,1.481C13.477,9.174,14.461,9,15.495,9 c1.036,0,2.024,0.174,2.922,0.483C18.675,9.17,19.763,8,21.565,8c0.732,0.731,0.381,2.656,0.102,3.594 c0.836,0.945,1.328,2.066,1.328,3.226c0,2.697-1.904,4.684-5.894,5.097C18.199,20.49,19,22.1,19,23.313v2.734 c0,0.104-0.023,0.179-0.035,0.268C23.641,24.676,27,20.236,27,15C27,8.373,21.627,3,15,3z"
          ></path>
        </svg>
      </a>
      {#if $currentFilePath}
        <span class="text-neutral-400 font-light text-sm mx-2">/</span>
        <span
          class="text-sm font-normal text-neutral-600 dark:text-neutral-300"
        >
          {$currentFilePath.split(/[\\/]/).pop()}
          {#if $isUnsaved}
            <span class="text-amber-500 font-bold ml-1" title="Unsaved changes"
              >*</span
            >
          {/if}
        </span>
      {/if}
    </div>
  </div>

  <!-- Actions -->
  <div class="flex flex-row justify-end items-center gap-4">
    <!-- Goal toggle: red / blue -->
    <button
      type="button"
      title={useRedGoal ? "Red goal (click for blue)" : "Blue goal (click for red)"}
      on:click={() => (useRedGoal = !useRedGoal)}
      class="px-3 py-1.5 text-sm font-semibold rounded border-2 transition-colors {useRedGoal
        ? 'bg-red-500/90 border-red-600 text-white hover:bg-red-600'
        : 'bg-blue-500/90 border-blue-600 text-white hover:bg-blue-600'}"
    >
      {useRedGoal ? "Red goal" : "Blue goal"}
    </button>

    <button
      type="button"
      title={snapToGoal ? "Snap to goal (θ_b extended 6\" in +x)" : "Snap to tag (θ_b to goal center)"}
      on:click={() => (snapToGoal = !snapToGoal)}
      class="px-3 py-1.5 text-sm font-medium rounded border-2 transition-colors {snapToGoal
        ? 'bg-amber-500/90 border-amber-600 text-white hover:bg-amber-600'
        : 'bg-neutral-200 dark:bg-neutral-600 border-neutral-300 dark:border-neutral-500 text-neutral-800 dark:text-neutral-200 hover:bg-neutral-300 dark:hover:bg-neutral-500'}"
    >
      {snapToGoal ? "Snap to goal" : "Snap to tag"}
    </button>

    <!-- Divider -->
    <div
      class="h-6 border-l border-neutral-300 dark:border-neutral-700 mx-4"
      aria-hidden="true"
    ></div>

    <!-- Snap to grid toggle -->
    {#if $showGrid}
      <button
        title={$snapToGrid ? "Disable Snap to Grid" : "Enable Snap to Grid"}
        on:click={() => snapToGrid.update((v) => !v)}
        class:text-green-500={$snapToGrid && $showGrid}
        class:text-gray-400={!$showGrid}
        class:opacity-50={!$showGrid}
        disabled={!$showGrid}
      >
        <svg
          xmlns="http://www.w3.org/2000/svg"
          width="24"
          height="24"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
          stroke-linecap="round"
          stroke-linejoin="round"
        >
          <!-- When snapped, show magnet icon -->
          <path
            d="m6 15-4-4 6.75-6.77a7.79 7.79 0 0 1 11 11L13 22l-4-4 6.39-6.36a2.14 2.14 0 0 0-3-3L6 15"
          ></path>
          <path d="m5 8 4 4"></path>
          <path d="m12 15 4 4"></path>

          <!-- If the snap is disabled, turn the icon grey, not white -->
          {#if !$snapToGrid}
            <line x1="23" y1="23" x2="1" y2="1"></line>
            class="opacity-50"
          {/if}
        </svg>
      </button>
    {/if}

    <!-- Grid toggle -->
    <button
      title={$showGrid ? `Grid: ${selectedGridSize}" (click to cycle)` : "Toggle Grid"}
      on:click={cycleGridSize}
      class:text-blue-500={$showGrid}
      class="relative"
    >
      <svg
        xmlns="http://www.w3.org/2000/svg"
        width="24"
        height="24"
        viewBox="0 0 24 24"
        fill="none"
        stroke="currentColor"
        stroke-width="2"
        stroke-linecap="round"
        stroke-linejoin="round"
      >
        <rect x="3" y="3" width="18" height="18" rx="2" ry="2"></rect>
        <line x1="3" y1="9" x2="21" y2="9"></line>
        <line x1="3" y1="15" x2="21" y2="15"></line>
        <line x1="9" y1="3" x2="9" y2="21"></line>
        <line x1="15" y1="3" x2="15" y2="21"></line>
      </svg>
      {#if $showGrid}
        <span class="absolute -bottom-5 left-1/2 -translate-x-1/2 text-xs font-semibold whitespace-nowrap">
          {selectedGridSize}"
        </span>
      {/if}
    </button>

    <!-- Ruler toggle -->
    <button
      title="Toggle Ruler"
      on:click={() => showRuler.update((v) => !v)}
      class:text-blue-500={$showRuler}
    >
      <svg
        xmlns="http://www.w3.org/2000/svg"
        width="24"
        height="24"
        viewBox="0 0 24 24"
        fill="none"
        stroke="currentColor"
        stroke-width="2"
        stroke-linecap="round"
        stroke-linejoin="round"
      >
        <path
          d="M21.3 15.3a2.4 2.4 0 0 1 0 3.4l-2.6 2.6a2.4 2.4 0 0 1-3.4 0L2.7 8.7a2.41 2.41 0 0 1 0-3.4l2.6-2.6a2.41 2.41 0 0 1 3.4 0z"
        ></path>
        <path d="m14.5 12.5 2-2"></path>
        <path d="m11.5 9.5 2-2"></path>
        <path d="m8.5 6.5 2-2"></path>
        <path d="m17.5 15.5 2-2"></path>
      </svg>
    </button>

    <!-- Protractor lock to robot toggle -->
    {#if $showProtractor}
      <button
        title={$protractorLockToRobot
          ? "Unlock Protractor from Robot"
          : "Lock Protractor to Robot"}
        on:click={() => protractorLockToRobot.update((v) => !v)}
        class:text-amber-500={$protractorLockToRobot}
      >
        {#if $protractorLockToRobot}
          <svg
            xmlns="http://www.w3.org/2000/svg"
            width="24"
            height="24"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <rect x="3" y="11" width="18" height="11" rx="2" ry="2"></rect>
            <path d="M7 11V7a5 5 0 0 1 10 0v4"></path>
          </svg>
        {:else}
          <svg
            xmlns="http://www.w3.org/2000/svg"
            width="24"
            height="24"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <rect x="3" y="11" width="18" height="11" rx="2" ry="2"></rect>
            <path d="M7 11V7a5 5 0 0 1 9.9-1"></path>
          </svg>
        {/if}
      </button>
    {/if}

    <!-- Protractor toggle -->

    <button
      title="Toggle Protractor"
      on:click={() => showProtractor.update((v) => !v)}
      class:text-blue-500={$showProtractor}
    >
      <svg
        xmlns="http://www.w3.org/2000/svg"
        width="24"
        height="24"
        viewBox="0 0 24 24"
        fill="none"
        stroke="currentColor"
        stroke-width="2"
        stroke-linecap="round"
        stroke-linejoin="round"
      >
        <path d="M12 21a9 9 0 1 1 0-18c2.52 0 4.93 1 6.74 2.74L21 8"></path>
        <path d="M12 3v6l3.7 2.7"></path>
      </svg>
    </button>

    <div
      class="h-6 border-l border-neutral-300 dark:border-neutral-700 mx-4"
      aria-hidden="true"
    ></div>

    <div class="flex items-center gap-3">
      <!-- Settings button -->
      <button title="Open Settings" on:click={() => (settingsOpen = true)}>
        <svg
          xmlns="http://www.w3.org/2000/svg"
          width="24"
          height="24"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
          stroke-linecap="round"
          stroke-linejoin="round"
          ><circle cx="12" cy="12" r="3"></circle><path
            d="M19.4 15a1.65 1.65 0 0 0 .33 1.82l.06.06a2 2 0 0 1 0 2.83 2 2 0 0 1-2.83 0l-.06-.06a1.65 1.65 0 0 0-1.82-.33 1.65 1.65 0 0 0-1 1.51V21a2 2 0 0 1-2 2 2 2 0 0 1-2-2v-.09A1.65 1.65 0 0 0 9 19.4a1.65 1.65 0 0 0-1.82.33l-.06.06a2 2 0 0 1-2.83 0 2 2 0 0 1 0-2.83l.06-.06a1.65 1.65 0 0 0 .33-1.82 1.65 1.65 0 0 0-1.51-1H3a2 2 0 0 1-2-2 2 2 0 0 1 2-2h.09A1.65 1.65 0 0 0 4.6 9a1.65 1.65 0 0 0-.33-1.82l-.06-.06a2 2 0 0 1 0-2.83 2 2 0 0 1 2.83 0l.06.06a1.65 1.65 0 0 0 1.82.33H9a1.65 1.65 0 0 0 1-1.51V3a2 2 0 0 1 2-2 2 2 0 0 1 2 2v.09a1.65 1.65 0 0 0 1 1.51 1.65 1.65 0 0 0 1.82-.33l.06-.06a2 2 0 0 1 2.83 0 2 2 0 0 1 0 2.83l-.06.06a1.65 1.65 0 0 0-.33 1.82V9a1.65 1.65 0 0 0 1.51 1H21a2 2 0 0 1 2 2 2 2 0 0 1-2 2h-.09a1.65 1.65 0 0 0-1.51 1z"
          ></path></svg
        >
      </button>
    </div>
  </div>
</div>

