<script lang="ts">
  import { now } from './now.svelte'
  import type { VehicleEntry, StopEntry } from './types'

  let {
    vehicles = [] as VehicleEntry[],
    allStops = new Map() as Map<string, StopEntry>,
    routeColor = '#677483',
    onRefresh = () => {},
  } = $props()

  let refreshTs = $state(Date.now())
  async function handleRefresh() {
    refreshTs = Date.now()
    await onRefresh()
  }

  function stopName(id: string | undefined | null): string {
    if (!id) return '—'
    return allStops.get(id)?.name || id
  }

  function etaSec(offset: number | undefined | null): string {
    if (offset == null) return '—'
    const abs = Math.abs(offset)
    const sign = offset < 0 ? '-' : ''
    if (abs < 60) return `${sign}${abs}s`
    return `${sign}${Math.floor(abs / 60)}m ${abs % 60}s`
  }

  function ageStr(ts: number | undefined | null, nowMs: number): string {
    if (!ts) return '—'
    const sec = Math.round((nowMs - ts) / 1000)
    if (sec < 3) return 'now'
    if (sec < 120) return `${sec}s ago`
    return `${Math.floor(sec / 60)}m ${sec % 60}s ago`
  }

  function devStr(sd: number | undefined | null): { text: string; cls: string } {
    if (sd == null) return { text: '—', cls: '' }
    const abs = Math.abs(sd)
    const unit = abs < 60 ? `${abs}s` : `${Math.floor(abs / 60)}m ${abs % 60}s`
    if (sd > 0) return { text: `${unit} late`, cls: 'late' }
    if (sd < 0) return { text: `${unit} early`, cls: 'early' }
    return { text: 'on time', cls: 'ontime' }
  }

  // Derive status from phase + GPS data
  function busStatus(v: VehicleEntry): { label: string; cls: string } {
    const phase = v.tripStatus?.phase || ''
    const hasGps = !!(v.location || v.tripStatus?.position)
    const hasRecentGps = hasGps && v.lastLocationUpdateTime && (Date.now() - v.lastLocationUpdateTime) < 600000

    if (phase === 'in_transit_to') return { label: 'Moving', cls: 'moving' }
    if (phase === 'stopped_at') return { label: 'At stop', cls: 'stopped' }
    if (phase === 'layover_departing') return { label: 'Departing', cls: 'moving' }
    if (phase === 'in_progress' && hasRecentGps) return { label: 'Moving', cls: 'moving' }
    if (phase === 'in_progress') return { label: 'Active', cls: 'other' }
    if (hasRecentGps) return { label: 'Active', cls: 'other' }
    if (hasGps) return { label: 'Stale GPS', cls: 'other' }
    return { label: 'Scheduled', cls: 'other' }
  }

  // Compact vehicle label: last 4 chars of vehicle ID
  function shortId(id: string): string {
    return id.length > 4 ? id.slice(-4) : id
  }

  let sorted = $derived(
    [...vehicles].sort((a, b) => {
      const order = { moving: 0, stopped: 1, other: 2 }
      const sa = order[busStatus(a).cls as keyof typeof order] ?? 2
      const sb = order[busStatus(b).cls as keyof typeof order] ?? 2
      return sa - sb
    })
  )
</script>

<div class="list-header">
  <span class="label-sm">{vehicles.length} bus{vehicles.length !== 1 ? 'es' : ''}</span>
  <div class="list-header-right">
    <span class="label-sm mono list-ts">{new Date(refreshTs).toLocaleTimeString()}</span>
    <button class="refresh-btn" onclick={handleRefresh} title="Refresh">⟳</button>
  </div>
</div>

{#if vehicles.length === 0}
  <div class="section-label label-sm dim">none</div>
{:else}
  <div class="v-stack">
    {#each sorted as v (v.vehicleId)}
      {@const bs = busStatus(v)}
      {@const dev = devStr(v.tripStatus?.scheduleDeviation)}
      {@const cs = v.tripStatus?.closestStop}
      {@const ns = v.tripStatus?.nextStop}
      {@const lu = v.tripStatus?.lastUpdateTime || v.lastUpdateTime}
      {@const cso = v.tripStatus?.closestStopTimeOffset != null && lu ? Math.round((lu + v.tripStatus.closestStopTimeOffset * 1000 - now.getTime()) / 1000) : null}
      {@const nso = v.tripStatus?.nextStopTimeOffset != null && lu ? Math.round((lu + v.tripStatus.nextStopTimeOffset * 1000 - now.getTime()) / 1000) : null}
      <div class="v-card" style="border-left-color: {routeColor}">
        <div class="v-head">
          <span class="v-dot {bs.cls}" title="phase={v.tripStatus?.phase || '—'}"></span>
          <span class="v-label mono">#{shortId(v.vehicleId)}</span>
          <span class="v-status {bs.cls}">{bs.label}</span>
          <span class="v-dev {dev.cls} mono">{dev.text}</span>
        </div>
        <div class="v-body">
          {#if ns && nso != null}
            <div class="v-row">
              <span class="v-row-label">next →</span>
              <span class="v-row-val">{stopName(ns)}</span>
              <span class="v-row-meta mono">{etaSec(nso)}</span>
            </div>
          {/if}
          {#if cs && cso != null && cs !== ns}
            <div class="v-row">
              <span class="v-row-label">near</span>
              <span class="v-row-val" class:dim={cso < 0}>{stopName(cs)}</span>
              <span class="v-row-meta mono" class:dim={cso < 0}>{etaSec(cso)}</span>
            </div>
          {/if}
          {#if !ns && !cs}
            <div class="v-row">
              <span class="v-row-label">next →</span>
              <span class="v-row-val dim">—</span>
            </div>
          {/if}

        </div>
        <div class="v-foot">
          <span class="v-age label-sm mono">as of {ageStr(lu, now.getTime())}</span>
        </div>
      </div>
    {/each}
  </div>
{/if}

<style>
  .list-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 2px 4px;
  }
  .list-header-right {
    display: flex;
    align-items: center;
    gap: 6px;
  }
  .list-ts {
    font-size: 10px !important;
    opacity: 0.5;
  }
  .refresh-btn {
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
  .refresh-btn:hover {
    background: var(--m3c-surface-container-high);
    color: var(--m3c-primary);
    border-color: var(--m3c-primary);
  }
  .section-label {
    padding: 2px 4px;
  }
  .section-label.dim {
    opacity: 0.4;
  }
  .v-stack {
    display: flex;
    flex-direction: column;
    gap: 6px;
  }
  .v-card {
    background: var(--m3c-surface-container);
    border-radius: 10px;
    border-left: 3px solid var(--m3c-outline);
    padding: 10px 12px;
    display: flex;
    flex-direction: column;
    gap: 6px;
  }
  .v-head {
    display: flex;
    align-items: center;
    gap: 6px;
  }
  .v-dot {
    width: 8px;
    height: 8px;
    border-radius: 50%;
    flex-shrink: 0;
  }
  .v-dot.moving { background: #2e7d32; }
  .v-dot.stopped { background: #d32f2f; }
  .v-dot.other { background: var(--m3c-on-surface-variant); opacity: 0.5; }
  .v-label {
    font-size: 13px;
    font-weight: 600;
    color: var(--m3c-on-surface);
  }
  .v-status {
    font-size: 10px;
    font-weight: 500;
    padding: 1px 6px;
    border-radius: 4px;
  }
  .v-status.moving { background: #e8f5e9; color: #1b5e20; }
  .v-status.stopped { background: #ffebee; color: #b71c1c; }
  .v-status.other { background: var(--m3c-surface-container-high); color: var(--m3c-on-surface-variant); }
  .v-dev {
    margin-left: auto;
    font-size: 11px;
    font-weight: 500;
  }
  .v-dev.late { color: var(--m3c-error); }
  .v-dev.early { color: var(--m3c-tertiary); }
  .v-dev.ontime { color: var(--m3c-on-surface-variant); }
  .v-body {
    display: flex;
    flex-direction: column;
    gap: 4px;
  }
  .v-row {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 12px;
  }
  .v-row-label {
    font-size: 10px;
    color: var(--m3c-on-surface-variant);
    font-weight: 500;
    width: 40px;
    flex-shrink: 0;
  }
  .v-row-val {
    flex: 1;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    font-weight: 500;
  }
  .v-row-val.dim {
    color: var(--m3c-on-surface-variant);
    opacity: 0.6;
  }
  .v-row-meta {
    font-size: 11px;
    font-weight: 600;
    color: var(--m3c-primary);
    flex-shrink: 0;
  }
  .v-foot {
    display: flex;
    justify-content: space-between;
    align-items: center;
  }
  .v-age {
    font-size: 10px !important;
    color: var(--m3c-on-surface-variant);
    opacity: 0.7;
  }
  @media (prefers-color-scheme: dark) {
    .v-status.moving { background: #1b3a1b; color: #a5d6a7; }
    .v-status.stopped { background: #3a1b1b; color: #ef9a9a; }
  }
</style>
