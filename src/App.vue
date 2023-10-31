<template>
  <div id="map"></div>
  <div id="snackbar" :class="{ show: copied }">Copied tile key to clipboard.</div>
</template>

<script setup>
import { Map } from 'maplibre-gl';
import tilebelt from '@mapbox/tilebelt';
import tc from '@mapbox/tile-cover';
import { onMounted } from 'vue';
import { useClipboard } from '@vueuse/core';

function update() {
  updateTiles();
}

function updateTiles() {
  const extentsGeom = getExtentsGeom();
  const zoom = Math.ceil(map.getZoom());
  const tiles = tc.tiles(extentsGeom, { min_zoom: zoom, max_zoom: zoom });

  map.getSource('tiles-geojson').setData({
    type: 'FeatureCollection',
    features: tiles.map((tile) => ({
      type: 'Feature',
      properties: {
        even: (tile[0] + tile[1]) % 2 == 0,
        quadkey: tilebelt.tileToQuadkey(tile),
        tile: JSON.stringify(tile),
      },
      geometry: tilebelt.tileToGeoJSON(tile),
    })),
  });

  map.getSource('tiles-centers-geojson').setData({
    type: 'FeatureCollection',
    features: tiles.map((tile) => {
      const box = tilebelt.tileToBBOX(tile);
      const center = [(box[0] + box[2]) / 2, (box[1] + box[3]) / 2];

      const quadkey = tilebelt.tileToQuadkey(tile);

      return {
        type: 'Feature',
        properties: {
          text:
            'Tile: ' +
            JSON.stringify(tile) +
            '\nQuadkey: ' +
            quadkey +
            '\nZoom: ' +
            tile[2],
          quadkey: quadkey,
        },
        geometry: {
          type: 'Point',
          coordinates: center,
        },
      };
    }),
  });
}

function getExtentsGeom() {
  const e = map.getBounds();
  const box = [
    e.getSouthWest().toArray(),
    e.getNorthWest().toArray(),
    e.getNorthEast().toArray(),
    e.getSouthEast().toArray(),
    e.getSouthWest().toArray(),
  ].map((coords) => {
    if (coords[0] < -180) return [-179.99999, coords[1]];
    if (coords[0] > 180) return [179.99999, coords[1]];
    return coords;
  });

  return {
    type: 'Polygon',
    coordinates: [box],
  };
}

const { copy, copied } = useClipboard();

let map;

onMounted(() => {
  map = new Map({
    container: 'map',
    style: {
      version: 8,
      sources: {
        'raster-tiles': {
          type: 'raster',
          tiles: [
            'https://basemaps.cartocdn.com/rastertiles/voyager/{z}/{x}/{y}@2x.png',
          ],
          attribution:
            '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors &copy; <a href="https://carto.com/attributions">CARTO</a>',
        },
      },
      layers: [
        {
          id: 'base',
          type: 'raster',
          source: 'raster-tiles',
        },
      ],
      glyphs: 'https://demotiles.maplibre.org/font/{fontstack}/{range}.pbf',
    },
    center: [0, 0],
    zoom: 2,
  });

  map.on('load', () => {
    map.addSource('tiles-geojson', {
      type: 'geojson',
      data: {
        type: 'FeatureCollection',
        features: [],
      },
    });

    map.addSource('tiles-centers-geojson', {
      type: 'geojson',
      data: {
        type: 'FeatureCollection',
        features: [],
      },
    });

    map.addLayer({
      id: 'tiles',
      source: 'tiles-geojson',
      type: 'line',
      paint: {
        'line-color': '#000',
      },
    });

    map.addLayer({
      id: 'tiles-shade',
      source: 'tiles-geojson',
      type: 'fill',
      paint: {
        'fill-color': [
          'case',
          ['get', 'even'],
          'rgba(0,0,0,0.1)',
          'rgba(0,0,0,0)',
        ],
      },
    });

    map.addLayer({
      id: 'tiles-centers',
      source: 'tiles-centers-geojson',
      type: 'symbol',
      layout: {
        'text-field': [
          'format',
          ['get', 'text'],
          {
            'font-scale': 1.2,
          },
        ],
        'text-offset': [0, -1],
      },
      paint: {
        'text-color': '#000',
        'text-color-transition': {
          duration: 0,
        },
        'text-halo-color': '#fff',
        'text-halo-width': 0.5,
      },
    });

    update();
  });

  map.on('moveend', update);

  map.on('click', (e) => {
    const features = map.queryRenderedFeatures(e.point, {
      layers: ['tiles-shade'],
    });

    copy(features[0].properties.tile);
  });
})
</script>