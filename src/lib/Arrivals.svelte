<script lang="ts">
  import { now } from './now.svelte'
  import { fetchArrivals, routeColor } from './api'
  import type { ArrivalDeparture, StopEntry } from './types'

  let {
    stop = null as StopEntry | null,
    routeId = '',
  } = $props()

  let refreshKey = $state(0)
  function refresh() { refreshKey++ }

  function etaSec(ts: number, nowMs: number): string {
    const sec = Math.round((ts - nowMs) / 1000)
    const abs = Math.abs(sec)
    if (abs < 60) return `${sec}s`
    const sign = sec < 0 ? '-' : ''
    const mins = Math.floor(abs / 60)
    const rem = abs % 60
    if (abs < 3600) return `${sign}${mins}m ${rem}s`
    return `${sign}${Math.floor(abs / 3600)}h ${Math.floor((abs % 3600) / 60)}m`
  }

  function delayStr(sched: number, pred: number): string {
    const sec = Math.round((pred - sched) / 1000)
    const abs = Math.abs(sec)
    const unit = abs < 60 ? `${abs}s` : `${Math.floor(abs / 60)}m ${abs % 60}s`
    if (sec > 0) return `${unit} late`
    if (sec < 0) return `${unit} early`
    return 'on time'
  }

  function delayClass(sched: number, pred: number): string {
    const sec = Math.round((pred - sched) / 1000)
    if (sec > 30) return 'late'
    if (sec < -30) return 'early'
    return 'ontime'
  }

  function ageStr(ts: number, nowMs: number): string {
    const sec = Math.round((nowMs - ts) / 1000)
    if (sec < 3) return 'now'
    if (sec < 120) return `${sec}s ago`
    return `${Math.floor(sec / 60)}m ${sec % 60}s ago`
  }

  function deduped(arrivals: ArrivalDeparture[]): ArrivalDeparture[] {
    // Filter to selected route only
    const filtered = routeId ? arrivals.filter(a => a.routeId === routeId) : arrivals
    const seen = new Set<string>()
    return filtered.filter(a => {
      const k = a.routeId + ':' + a.tripId
      if (seen.has(k)) return false; seen.add(k); return true
    }).sort((a, b) =>
      (a.predictedArrivalTime || a.scheduledArrivalTime || 0)
      - (b.predictedArrivalTime || b.scheduledArrivalTime || 0)
    ).slice(0, 20)
  }

  function hasRealTime(a: ArrivalDeparture): boolean {
    return a.predicted && !!a.predictedArrivalTime && !!a.lastUpdateTime
  }
</script>

{#if !stop}
  <div class="section-label label-sm">tap a stop</div>
{:else}
  {#key stop.id + ':' + refreshKey}
    {#await fetchArrivals(stop.id)}
      <div class="section-label label-sm">loading…</div>
    {:then arrivals}
      {@const dedupedArrivals = deduped(arrivals)}
      {#if dedupedArrivals.length === 0}
        <div class="section-label label-sm">none for this route</div>
      {:else}
        <div class="a-header">
          <span class="label-sm">{dedupedArrivals.length} arrival{dedupedArrivals.length !== 1 ? 's' : ''}</span>
          <button class="a-refresh" onclick={refresh} title="Refresh arrivals">⟳</button>
        </div>
        <div class="a-stack">
          {#each dedupedArrivals as a (a.tripId + a.stopId)}
            {@const rt = hasRealTime(a)}
            <div class="a-card" class:rt>
              <div class="a-head">
                <span class="a-route" style="background:{routeColor(a.routeId)}">
                  {a.routeShortName || a.routeId.split('_').pop()}
                </span>
                <span class="a-dest">{a.tripHeadsign || a.routeLongName || '—'}</span>
              </div>

              <div class="a-body">
                {#if rt}
                  {#if a.predictedArrivalTime}
                    <div class="a-eta mono">{etaSec(a.predictedArrivalTime, now.getTime())}</div>
                  {/if}
                  <div class="a-meta">
                    {#if a.scheduledArrivalTime && a.predictedArrivalTime}
                      <span class="a-delay {delayClass(a.scheduledArrivalTime, a.predictedArrivalTime)} mono">
                        {delayStr(a.scheduledArrivalTime, a.predictedArrivalTime)}
                      </span>
                      <span class="a-sep">·</span>
                    {/if}
                    <span class="a-age mono">as of {ageStr(a.lastUpdateTime, now.getTime())}</span>
                  </div>
                {:else}
                  {#if a.scheduledArrivalTime}
                    <div class="a-eta mono scheduled-eta">{etaSec(a.scheduledArrivalTime, now.getTime())}</div>
                  {/if}
                  <div class="a-meta">
                    <span class="a-sched-label">timetable</span>
                  </div>
                {/if}
              </div>


            </div>
          {/each}
        </div>
      {/if}
    {:catch error}
      <div class="a-error">⚠ failed <button class="a-refresh" onclick={refresh}>⟳</button></div>
    {/await}
  {/key}
{/if}

<style>
  .section-label {
    padding: 2px 4px;
  }
  .a-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 2px 4px;
  }
  .a-refresh {
    background: none;
    border: 1px solid var(--m3c-outline-variant);
    border-radius: 6px;
    color: var(--m3c-on-surface-variant);
    font-size: 14px;
    line-height: 1;
    padding: 2px 6px;
    cursor: pointer;
    transition: all 0.1s;
  }
  .a-refresh:hover {
    background: var(--m3c-surface-container-high);
    color: var(--m3c-primary);
    border-color: var(--m3c-primary);
  }
  .a-stack {
    display: flex;
    flex-direction: column;
    gap: 6px;
  }
  .a-card {
    background: var(--m3c-surface-container);
    border-radius: 10px;
    padding: 10px 12px;
    display: flex;
    flex-direction: column;
    gap: 5px;
    border: 1px solid transparent;
  }
  .a-card.rt {
    border-color: var(--m3c-primary-container-subtle);
  }
  .a-head {
    display: flex;
    align-items: center;
    gap: 8px;
  }
  .a-route {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    min-width: 28px;
    height: 20px;
    border-radius: 4px;
    color: #fff;
    font-weight: 700;
    font-size: 10px;
    padding: 0 5px;
    flex-shrink: 0;
  }
  .a-dest {
    font-size: 13px;
    font-weight: 500;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
  .a-body {
    display: flex;
    align-items: baseline;
    gap: 8px;
  }
  .a-eta {
    font-size: 18px;
    font-weight: 600;
    color: var(--m3c-primary);
    line-height: 1;
  }
  .a-eta.scheduled-eta {
    color: var(--m3c-on-surface-variant);
    opacity: 0.7;
  }
  .a-meta {
    display: flex;
    align-items: center;
    gap: 4px;
    font-size: 11px;
  }
  .a-delay.late { color: var(--m3c-error); font-weight: 500; }
  .a-delay.early { color: var(--m3c-tertiary); font-weight: 500; }
  .a-delay.ontime { color: var(--m3c-on-surface-variant); }
  .a-sep { color: var(--m3c-outline); font-size: 10px; }
  .a-age { color: var(--m3c-on-surface-variant); opacity: 0.7; }
  .a-sched-label {
    font-size: 10px;
    color: var(--m3c-on-surface-variant);
    opacity: 0.6;
  }
  .a-error {
    color: var(--m3c-error);
    font-size: 12px;
    padding: 6px 8px;
    background: var(--m3c-error-container-subtle);
    border-radius: 8px;
  }
</style>
