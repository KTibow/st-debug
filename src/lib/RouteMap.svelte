<script lang="ts">
  import { onMount, onDestroy } from 'svelte'
  import L from 'leaflet'
  import 'leaflet/dist/leaflet.css'
  import { decodePolyline } from './api'
  import type { StopEntry, VehicleEntry } from './types'

  let {
    stops = [] as StopEntry[],
    vehicles = [] as VehicleEntry[],
    routeColor = '#677483',
    polylines = [] as string[],
    stopId = null as string | null,
    onStopClick = (_id: string) => {},
  } = $props()

  let mapContainer: HTMLDivElement
  let map: L.Map
  let vehicleLayer = L.layerGroup()
  let routeLayer = L.layerGroup()
  let highlightLayer = L.layerGroup()

  onMount(() => {
    map = L.map(mapContainer, { zoomControl: true, attributionControl: false }).setView([47.6, -122.2], 10)
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', { maxAge: 60000 }).addTo(map)
    vehicleLayer.addTo(map)
    routeLayer.addTo(map)
    highlightLayer.addTo(map)
  })

  onDestroy(() => { map?.remove() })

  // Route shape: stops + polylines
  $effect(() => {
    if (!map) return
    routeLayer.clearLayers()
    for (const enc of polylines) {
      const pts = decodePolyline(enc)
      if (pts.length > 1) {
        L.polyline(pts, { color: routeColor, weight: 3, opacity: 0.55 }).addTo(routeLayer)
      }
    }
    for (const s of stops) {
      L.circleMarker([s.lat, s.lon], {
        radius: 4, color: '#fff', fillColor: routeColor, fillOpacity: 0.85, weight: 2
      }).addTo(routeLayer)
        .bindPopup(`<b>${s.name}</b>`)
        .on('click', () => onStopClick(s.id))
    }
    if (stops.length > 0) {
      const bounds = L.latLngBounds(stops.map(s => [s.lat, s.lon]))
      for (const enc of polylines) decodePolyline(enc).forEach(p => bounds.extend(p))
      if (bounds.isValid()) map.fitBounds(bounds, { padding: [60, 60] })
    }
  })

  // Vehicles layer
  $effect(() => {
    if (!map) return
    vehicleLayer.clearLayers()
    for (const v of vehicles) {
      const pos = v.location || v.tripStatus?.position
      if (!pos || !pos.lat || !pos.lon) continue
      const moving = v.status === 'IN_TRANSIT_TO' || v.tripStatus?.phase === 'in_transit_to'
      const stopped = v.status === 'STOPPED_AT' || v.tripStatus?.phase === 'stopped_at'
      L.circleMarker([pos.lat, pos.lon], {
        radius: 6, fillColor: moving ? '#2e7d32' : stopped ? '#d32f2f' : '#bbb',
        color: '#fff', weight: 1.5, fillOpacity: 0.85
      }).addTo(vehicleLayer).bindPopup(
        `<b>Bus #${v.vehicleId.slice(-4)}</b><br/>${v.status || '—'}`
      )
    }
  })

  // Highlighted stop
  $effect(() => {
    if (!map) return
    highlightLayer.clearLayers()
    if (stopId) {
      const s = stops.find(s => s.id === stopId)
      if (s) {
        L.circle([s.lat, s.lon], { radius: 90, color: routeColor, fillColor: routeColor, fillOpacity: 0.08, weight: 2 }).addTo(highlightLayer)
        map.setView([s.lat, s.lon], 14)
      }
    }
  })
</script>

<div bind:this={mapContainer} style="width:100%;height:100%"></div>
