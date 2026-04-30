<script lang="ts">
  import '../../app.css';
  import { onMount } from 'svelte';
  import {
    getTimerState,
    getSettings,
    getThemes,
    onTimerTick,
    onTimerPaused,
    onTimerResumed,
    onRoundChange,
    onTimerReset,
    timerToggle,
    timerReset,
    timerSkip,
  } from '$lib/ipc';
  import { applyTheme } from '$lib/stores/theme';
  import { resolveThemeName } from '$lib/utils/theme';
  import WidgetTimer from '$lib/components/WidgetTimer.svelte';
  import type { TimerState, Settings } from '$lib/types';
  import type { UnlistenFn } from '@tauri-apps/api/event';

  // Widget-local state (no shared stores — this window is independent)
  let timerState = $state<TimerState>({
    round_type: 'work',
    previous_round_type: '',
    elapsed_secs: 0,
    total_secs: 25 * 60,
    is_running: false,
    is_paused: false,
    work_round_number: 1,
    work_rounds_total: 4,
    session_work_count: 1,
  });

  let isHovered = $state(false);

  onMount(() => {
    const cleanups: UnlistenFn[] = [];

    (async () => {
      // Load initial timer state
      const initial = await getTimerState();
      timerState = initial;

      // Load and apply theme (same logic as main window)
      const s: Settings = await getSettings();
      const themes = await getThemes();
      const osDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
      const active = themes.find((t) => t.name === resolveThemeName(s, osDark)) ?? themes[0];
      if (active) applyTheme(active);

      // Subscribe to timer events
      cleanups.push(
        await onTimerTick(({ elapsed_secs, total_secs }) => {
          timerState = {
            ...timerState,
            elapsed_secs,
            total_secs,
            is_running: true,
            is_paused: false,
          };
        }),
        await onTimerPaused(({ elapsed_secs }) => {
          timerState = {
            ...timerState,
            elapsed_secs,
            is_running: false,
            is_paused: true,
          };
        }),
        await onTimerResumed(({ elapsed_secs }) => {
          timerState = {
            ...timerState,
            elapsed_secs,
            is_running: true,
            is_paused: false,
          };
        }),
        await onRoundChange((snap) => {
          timerState = snap;
        }),
        await onTimerReset((snap) => {
          timerState = snap;
        }),
      );

      // Listen for settings changes to update theme
      cleanups.push(
        await (await import('@tauri-apps/api/event')).listen<Settings>('settings:changed', async (e) => {
          const updated = e.payload;
          const allThemes = await getThemes();
          const dark = window.matchMedia('(prefers-color-scheme: dark)').matches;
          const t = allThemes.find((th) => th.name === resolveThemeName(updated, dark));
          if (t) applyTheme(t);
        }),
      );
    })();

    return () => {
      for (const fn of cleanups) fn();
    };
  });
</script>

<svelte:window
  onmouseenter={() => (isHovered = true)}
  onmouseleave={() => (isHovered = false)}
/>

<div class="widget-page" data-tauri-drag-region>
  <WidgetTimer
    {timerState}
    {isHovered}
    onToggle={timerToggle}
    onReset={timerReset}
    onSkip={timerSkip}
  />
</div>

<style>
  :global(body) {
    margin: 0;
    padding: 0;
    overflow: hidden;
    background: transparent !important;
    /* Prevent text selection during drag */
    user-select: none;
    -webkit-user-select: none;
  }

  .widget-page {
    width: 140px;
    height: 140px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: transparent;
  }
</style>
