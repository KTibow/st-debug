<script lang="ts">
  import RoutePicker from './lib/RoutePicker.svelte'
  import RouteMap from './lib/RouteMap.svelte'
  import VehicleList from './lib/VehicleList.svelte'
  import Arrivals from './lib/Arrivals.svelte'
  import { fetchRouteShape, fetchTripIdsForRoute, fetchAllVehicles, routeColor } from './lib/api'
  import type { StopEntry, VehicleEntry } from './lib/types'

  let routeId = $state('')
  let color = $state('#677483')
  let polylines: string[] = $state([])
  let stops: StopEntry[] = $state([])
  let allVehicles: VehicleEntry[] = $state([])
  let selectedStop: StopEntry | null = $state(null)
  let statusMsg = $state('')

  // All stops from the route shape response (includes lat/lon/name — no extra API calls)
  let allStops = $state<Map<string, StopEntry>>(new Map())

  // Direction state
  let directions: { name: string; stopIds: string[] }[] = $state([])
  let selectedDir = $state(-1)
  let tripIds = $state<Set<string>>(new Set())

  let refreshTimer: ReturnType<typeof setInterval> | null = null
  let showMap = $state(false)

  // Filter vehicles by direction + tripId
  let vehicles = $derived.by(() => {
    if (selectedDir < 0 || !directions[selectedDir]) return allVehicles
    const dirStops = new Set(directions[selectedDir].stopIds)
    return allVehicles.filter(v => {
      if (!v.tripId || !tripIds.has(v.tripId)) return false
      const cs = v.tripStatus?.closestStop
      const ns = v.tripStatus?.nextStop
      return (cs && dirStops.has(cs)) || (ns && dirStops.has(ns))
    })
  })

  async function selectRoute(id: string) {
    routeId = id
    stops = []; polylines = []; allVehicles = []; selectedStop = null
    selectedDir = -1; directions = []; tripIds = new Set()
    if (refreshTimer) { clearInterval(refreshTimer); refreshTimer = null }
    if (!id) return

    color = routeColor(id)
    statusMsg = '…'

    const [shapeResult, trips] = await Promise.all([
      fetchRouteShape(id),
      fetchTripIdsForRoute(id)
    ])
    if (!shapeResult) { statusMsg = 'failed to load'; return }
    tripIds = trips

    const shape = shapeResult.shape
    allStops = new Map(shapeResult.stops.map(s => [s.id, s]))

    polylines = (shape.polylines || []).map(pl => pl.points).filter(Boolean)

    const groups = shape.stopGroupings?.[0]?.stopGroups || []
    directions = groups.map(g => ({
      name: g.name?.name || g.name || 'Direction',
      stopIds: g.stopIds || []
    }))

    statusMsg = ''
  }

  async function selectDirection(idx: number) {
    selectedDir = idx
    if (idx < 0 || !directions[idx]) { stops = []; return }

    const ids = new Set(directions[idx].stopIds)
    stops = [...allStops.values()].filter(s => ids.has(s.id))

    await refreshVehicles()
    if (refreshTimer) clearInterval(refreshTimer)
    refreshTimer = setInterval(refreshVehicles, 15000)

    statusMsg = ''
  }

  async function refreshVehicles() {
    try {
      allVehicles = (await fetchAllVehicles()).filter(v => v.tripId && tripIds.has(v.tripId))
    } catch (e) { console.error(e) }
  }

  function handleStopClick(stopId: string) {
    selectedStop = stops.find(s => s.id === stopId) || null
  }
</script>

<div id="shell">
  <div id="data-panel">
    <RoutePicker onSelect={selectRoute} />

    {#if routeId}
      {#if directions.length > 0}
        <div class="dir-bar">
          {#each directions as dir, i}
            <button
              class="dir-chip"
              class:active={selectedDir === i}
              onclick={() => selectDirection(i)}
            >
              {dir.name}
              <span class="dir-count">{dir.stopIds.length}</span>
            </button>
          {/each}
        </div>
      {/if}

      {#if selectedDir >= 0}
        <div class="card-area">
          <VehicleList {vehicles} {allStops} routeColor={color} onRefresh={refreshVehicles} />
          <Arrivals stop={selectedStop} routeId={routeId} />
        </div>
      {:else}
        <div class="empty-panel">
          <div class="empty-icon">→</div>
          <p>Pick a direction</p>
        </div>
      {/if}

      {#if statusMsg}
        <div class="status-bar label-sm">{statusMsg}</div>
      {/if}
    {:else}
      <div class="empty-panel">
        <div class="empty-icon">🚍</div>
        <p>Pick a route &amp; direction</p>
      </div>
    {/if}
  </div>

  <button class="map-toggle" onclick={() => showMap = !showMap}>
    {showMap ? '✕' : '🗺'}
  </button>

  <div id="map-wrap" class:shown={showMap}>
    {#if routeId && selectedDir >= 0}
      <RouteMap {stops} {vehicles} routeColor={color} {polylines} stopId={selectedStop?.id ?? null} onStopClick={handleStopClick} />
    {:else}
      <div class="map-placeholder">
        <div class="empty-icon">🚍</div>
      </div>
    {/if}
  </div>
</div>

<style>
  #shell { display: flex; height: 100dvh; }
  #data-panel {
    width: 380px;
    min-width: 380px;
    background: var(--m3c-surface-container-low);
    display: flex;
    flex-direction: column;
    overflow-y: auto;
    border-right: 1px solid var(--m3c-outline-variant);
  }
  #map-wrap { flex: 1; position: relative; }

  /* Direction bar */
  .dir-bar {
    display: flex;
    gap: 4px;
    padding: 8px 12px;
    border-bottom: 1px solid var(--m3c-outline-variant);
    flex-wrap: wrap;
  }
  .dir-chip {
    padding: 6px 14px;
    border: 1px solid var(--m3c-outline);
    border-radius: 20px;
    background: transparent;
    color: var(--m3c-on-surface-variant);
    font-family: inherit;
    font-size: 12px;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.12s;
    display: inline-flex;
    align-items: center;
    gap: 4px;
  }
  .dir-chip:hover {
    border-color: var(--m3c-primary);
    color: var(--m3c-primary);
    background: var(--m3c-primary-container-subtle);
  }
  .dir-chip.active {
    background: var(--m3c-primary);
    color: var(--m3c-on-primary);
    border-color: var(--m3c-primary);
  }
  .dir-count {
    font-size: 10px;
    opacity: 0.7;
    font-weight: 400;
  }

  /* Card scroll area */
  .card-area {
    flex: 1;
    overflow-y: auto;
    padding: 8px 12px 12px;
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  /* Status bar */
  .status-bar {
    padding: 6px 12px;
    border-top: 1px solid var(--m3c-outline-variant);
    background: var(--m3c-surface-container);
    text-align: center;
    opacity: 0.6;
  }

  /* Empty states */
  .empty-panel {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 8px;
    color: var(--m3c-on-surface-variant);
    padding: 40px 20px;
    text-align: center;
  }
  .empty-icon { font-size: 36px; opacity: 0.4; }
  .empty-panel p { font-size: 13px; font-weight: 500; }
  .map-placeholder {
    width: 100%;
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
    background: var(--m3c-surface-container-low);
  }

  /* Map toggle */
  .map-toggle {
    display: none;
    position: fixed;
    bottom: 12px;
    right: 12px;
    z-index: 1000;
    width: 44px;
    height: 44px;
    border-radius: 50%;
    background: var(--m3c-surface-container-high);
    border: 1px solid var(--m3c-outline-variant);
    color: var(--m3c-on-surface-variant);
    font-size: 18px;
    cursor: pointer;
    box-shadow: 0 2px 8px rgba(0,0,0,0.15);
    transition: all 0.15s;
  }
  .map-toggle:hover {
    background: var(--m3c-surface-container-highest);
    transform: scale(1.05);
  }

  @media (width < 768px) {
    #shell { flex-direction: column; }
    #data-panel {
      width: 100vw;
      min-width: 0;
      max-height: 100dvh;
      border-right: none;
    }
    #map-wrap {
      display: none;
      position: fixed;
      inset: 0;
      z-index: 999;
    }
    #map-wrap.shown {
      display: block;
    }
    .map-toggle {
      display: flex;
      align-items: center;
      justify-content: center;
    }
  }
</style>
