<template>
  <div ref="mapContainer" class="w-full h-full" />
</template>

<script setup>
import { onMounted, ref, watch } from 'vue'
import maplibregl from 'maplibre-gl'

const props = defineProps({
  layers: Array,
  layerStates: Object,
  theme: String
})

const mapContainer = ref(null)
let map = null
let popup = null

function getTileURL(theme) {
  return theme === 'dark'
    ? 'https://basemaps.cartocdn.com/dark_all/{z}/{x}/{y}.png'
    : 'https://tile.openstreetmap.org/{z}/{x}/{y}.png'
}

onMounted(() => {
  map = new maplibregl.Map({
  container: mapContainer.value,
  style: {
    version: 8,
    sources: {},
    layers: [
      {
        id: 'background',
        type: 'background',
        paint: {
          'background-color': '#518151'
        }
      }
    ]
  },
  center: [0, 0],
  zoom: 13
})

  map.addControl(new maplibregl.NavigationControl(), 'top-right')
})

watch(() => props.theme, (newTheme) => {
  const tileURL = getTileURL(newTheme)
  if (map?.getSource('basemap')) {
    map.removeLayer('basemap')
    map.removeSource('basemap')

    map.addSource('basemap', {
      type: 'raster',
      tiles: [tileURL],
      tileSize: 256
    })

    map.addLayer({
      id: 'basemap',
      type: 'raster',
      source: 'basemap'
    }, map.getStyle().layers[0]?.id)
  }
})

watch(() => props.layers, (layers) => {
  if (!map || !layers?.length) return

  // Clean up previous dynamic layers
  map.getStyle().layers?.forEach(layer => {
    if (layer.id.startsWith('cs2-')) {
      if (map.getLayer(layer.id)) map.removeLayer(layer.id)
      if (map.getSource(layer.id)) map.removeSource(layer.id)
    }
  })

  const bounds = new maplibregl.LngLatBounds()

  for (const { id, data } of layers) {
    const layerId = `cs2-${id}`
    const style = props.layerStates?.[id]
    if (!style) continue

    const geomType = data.features?.[0]?.geometry?.type

    map.addSource(layerId, {
      type: 'geojson',
      data
    })

    let layerConfig = {
      id: layerId,
      type: geomType === 'Point' ? 'circle' : 'fill',
      source: layerId,
      layout: {
        visibility: style.visible ? 'visible' : 'none'
      },
      paint: {}
    }

    if (geomType === 'Point') {
      layerConfig.paint = {
        'circle-color': style.color,
        'circle-radius': 6
      }

      // Add POI popup
      map.on('click', layerId, (e) => {
        const feature = e.features[0]
        const props = feature?.properties
        const content = Object.entries(props ?? {})
          .map(([k, v]) => `<strong>${k}:</strong> ${v}`)
          .join('<br>')

        if (!popup) popup = new maplibregl.Popup({ closeOnClick: true })
        popup.setLngLat(e.lngLat).setHTML(content).addTo(map)
      })

      map.on('mouseenter', layerId, () => map.getCanvas().style.cursor = 'pointer')
      map.on('mouseleave', layerId, () => map.getCanvas().style.cursor = '')
    } else if (geomType?.includes('Line')) {
      layerConfig.type = 'line'
      layerConfig.paint = {
        'line-color': style.color,
        'line-width': 2
      }
    } else {
      layerConfig.type = 'fill'
      layerConfig.paint = {
        'fill-color': style.color,
        'fill-opacity': 0.5
      }
    }

    map.addLayer(layerConfig)

    data.features?.forEach(f => {
      const coords = f.geometry?.coordinates?.flat(Infinity)
      for (let i = 0; i < coords.length - 1; i += 2) {
        bounds.extend([coords[i], coords[i + 1]])
      }
    })
  }

  if (!bounds.isEmpty()) {
    map.fitBounds(bounds, { padding: 40 })
  }
}, { immediate: true, deep: true })

watch(() => props.layerStates, (vis) => {
  for (const [id, config] of Object.entries(vis)) {
    const layerId = `cs2-${id}`
    if (!map.getLayer(layerId)) continue

    map.setLayoutProperty(layerId, 'visibility', config.visible ? 'visible' : 'none')

    const type = map.getLayer(layerId)?.type
    if (type === 'fill') map.setPaintProperty(layerId, 'fill-color', config.color)
    if (type === 'line') map.setPaintProperty(layerId, 'line-color', config.color)
    if (type === 'circle') map.setPaintProperty(layerId, 'circle-color', config.color)
  }
}, { deep: true })
</script>
