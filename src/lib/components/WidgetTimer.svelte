<script lang="ts">
  import { tweened } from 'svelte/motion';
  import { cubicOut } from 'svelte/easing';
  import { fade } from 'svelte/transition';
  import type { TimerState } from '$lib/types';

  interface Props {
    timerState: TimerState;
    isHovered: boolean;
    onToggle: () => void;
    onReset: () => void;
    onSkip: () => void;
  }

  let { timerState, isHovered, onToggle, onReset, onSkip }: Props = $props();

  // SVG constants for the small widget dial
  const RADIUS = 50;
  const CIRCUMFERENCE = 2 * Math.PI * RADIUS; // ~314.16
  const CENTER = 70;
  const STROKE_WIDTH = 6;

  // Tweened dash offset for smooth animation
  const dashOffset = tweened(CIRCUMFERENCE, { duration: 800, easing: cubicOut });

  // Track previous round for snap animation on round change
  let prevRound = $state<string>('');

  function strokeColor(rt: string): string {
    if (rt === 'work') return 'var(--color-focus-round)';
    if (rt === 'short-break') return 'var(--color-short-round)';
    return 'var(--color-long-round)';
  }

  // Remaining time in MM:SS format
  function remainingTime(): string {
    const remaining = Math.max(0, timerState.total_secs - timerState.elapsed_secs);
    const mins = Math.floor(remaining / 60);
    const secs = remaining % 60;
    return `${mins}:${secs.toString().padStart(2, '0')}`;
  }

  $effect(() => {
    const progress = timerState.total_secs > 0 ? timerState.elapsed_secs / timerState.total_secs : 0;
    const target = CIRCUMFERENCE * (1 - progress);

    if (timerState.round_type !== prevRound) {
      dashOffset.set(CIRCUMFERENCE, { duration: 0 });
      prevRound = timerState.round_type;
    } else {
      dashOffset.set(target);
    }
  });
</script>

<div class="widget-timer" data-tauri-drag-region>
  <svg class="dial" viewBox="0 0 140 140" aria-hidden="true">
    <!-- Background track -->
    <circle
      cx={CENTER}
      cy={CENTER}
      r={RADIUS}
      fill="none"
      stroke="rgba(255, 255, 255, 0.15)"
      stroke-width={STROKE_WIDTH}
    />
    <!-- Progress arc -->
    <circle
      class="progress"
      cx={CENTER}
      cy={CENTER}
      r={RADIUS}
      fill="none"
      stroke={strokeColor(timerState.round_type)}
      stroke-width={STROKE_WIDTH}
      stroke-linecap="round"
      stroke-dasharray={CIRCUMFERENCE}
      stroke-dashoffset={$dashOffset}
      transform="rotate(-90 {CENTER} {CENTER})"
    />
  </svg>

  <!-- Time display centered on the dial -->
  <div class="time-display" data-tauri-drag-region>
    {remainingTime()}
  </div>

  <!-- Controls overlay (visible on hover) -->
  {#if isHovered}
    <div class="controls-overlay" transition:fade={{ duration: 150 }}>
      <!-- Pause / Play -->
      <button
        class="ctrl-btn ctrl-center"
        onclick={(e) => { e.stopPropagation(); onToggle(); }}
        aria-label={timerState.is_running ? 'Pause' : 'Play'}
      >
        {#if timerState.is_running}
          <svg width="14" height="14" viewBox="0 0 14 14">
            <rect x="1" y="0" width="4" height="14" rx="1" fill="currentColor" />
            <rect x="9" y="0" width="4" height="14" rx="1" fill="currentColor" />
          </svg>
        {:else}
          <svg width="14" height="14" viewBox="0 0 14 14">
            <polygon points="2,0 14,7 2,14" fill="currentColor" />
          </svg>
        {/if}
      </button>

      <!-- Reset (left) -->
      <button
        class="ctrl-btn ctrl-left"
        onclick={(e) => { e.stopPropagation(); onReset(); }}
        aria-label="Reset"
      >
        <svg width="12" height="12" viewBox="0 0 14 14">
          <polygon points="13,1 5,7 13,13" fill="currentColor" />
          <rect x="1" y="1" width="2.5" height="12" rx="1" fill="currentColor" />
        </svg>
      </button>

      <!-- Skip (right) -->
      <button
        class="ctrl-btn ctrl-right"
        onclick={(e) => { e.stopPropagation(); onSkip(); }}
        aria-label="Skip"
      >
        <svg width="12" height="12" viewBox="0 0 14 14">
          <polygon points="1,1 9,7 1,13" fill="currentColor" />
          <rect x="10.5" y="1" width="2.5" height="12" rx="1" fill="currentColor" />
        </svg>
      </button>
    </div>
  {/if}
</div>

<style>
  .widget-timer {
    position: relative;
    width: 140px;
    height: 140px;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .dial {
    position: absolute;
    width: 120px;
    height: 120px;
  }

  .progress {
    transition: stroke 0.3s ease;
  }

  .time-display {
    position: relative;
    z-index: 1;
    font-family: 'Mona Sans Mono', monospace;
    font-size: 1.1rem;
    font-weight: 600;
    color: var(--color-foreground, #e0e0e0);
    text-shadow: 0 1px 3px rgba(0, 0, 0, 0.5);
    letter-spacing: 0.05em;
    pointer-events: none;
  }

  /* Controls overlay */
  .controls-overlay {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 6px;
    z-index: 10;
  }

  .ctrl-btn {
    background: rgba(0, 0, 0, 0.5);
    border: 1px solid rgba(255, 255, 255, 0.2);
    color: var(--color-foreground, #e0e0e0);
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 50%;
    transition: background 0.15s ease, border-color 0.15s ease;
    padding: 0;
  }

  .ctrl-btn:hover {
    background: rgba(255, 255, 255, 0.15);
    border-color: rgba(255, 255, 255, 0.4);
  }

  .ctrl-center {
    width: 32px;
    height: 32px;
  }

  .ctrl-left,
  .ctrl-right {
    width: 26px;
    height: 26px;
  }
</style>
