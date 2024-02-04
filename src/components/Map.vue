<script setup>
'use strict';

/**
 * @author Boris Steiner
 */

import {FLIGHT1DATA, FLIGHT2DATA, MAPS1DATA, MAPS2DATA, MAPS3DATA, RUN1DATA} from './coords.js';

const dataSets = {
  FLIGHT1DATA,
  FLIGHT2DATA,
  MAPS1DATA,
  MAPS2DATA,
  MAPS3DATA,
  RUN1DATA
};

import 'mapbox-gl/dist/mapbox-gl.css';

import mapboxgl from 'mapbox-gl';
import LegendControl from 'mapboxgl-legend';
import '../../node_modules/mapboxgl-legend/dist/style.css';

import chroma from 'chroma-js';
import {onMounted, watch, onBeforeMount, ref} from "vue";

import create3dModelLayer from "./3dModelLayer.js";
import {state, stateKeys} from './showModelState.js';

document.addEventListener('DOMContentLoaded', async function () {
  // Dynamically load the windowToggle3d.js script
  await import('./windowToggle3d.js');

});
let allData = [];
onBeforeMount(() => {
  for (let data of Object.values(dataSets)) {
    allData = allData.concat(data);
  }
});

let isTerrainEnabled = ref(false);

function toggleTerrain() {
  isTerrainEnabled.value = !isTerrainEnabled.value;
}

let isSatelliteView = ref(false);

const toggleMapView = () => {
  isSatelliteView.value = !isSatelliteView.value;
}

onMounted(() => {
// Boris's token
  mapboxgl.accessToken = 'pk.eyJ1IjoidGhlb2xjaGkiLCJhIjoiY2xudzRicjhyMDZpeDJ0bzZoMmp5MHFqZSJ9.hsNF-iGKdszPYr1b0TXTcA';

  const map = new mapboxgl.Map({
    container: 'map',
    // style: 'mapbox://styles/theolchi/clnw50m2r003z01pg94do2adn', // satellite
    // style: 'mapbox://styles/theolchi/clorauxl600nc01qy4goi6wes', // streets
    // style: 'mapbox://styles/mapbox/satellite-streets-v12', // no idea
    style: 'mapbox://styles/mapbox/outdoors-v11', // different streets
    center: [calculateMeanValues(allData)[0], calculateMeanValues(allData)[1]],
    zoom: 15,
    antialias: true // create the gl context with MSAA antialiasing, so custom layers are antialiased
  });

  const legend = new LegendControl();
  map.addControl(legend, 'bottom-left');

  let usedColors = new Set();

  function getRandomUniqueHexColorWithInterval(interval) {
    let hue;
    do {
      hue = Math.floor(Math.random() * 360);
    } while (usedColors.has(hue));

    usedColors.add(hue);

    // Adjust hue to ensure visual distinction
    const adjustedHue = (hue + interval) % 360;
    const adjustedColor = chroma.hsl(adjustedHue, 1, 0.5).hex();

    return adjustedColor;
  }

  let currentlyVisiblePath = null;
  let endMarker = null;

  function addPath(map, id, coords) {
    // Generate a unique color for the path
    let pathColor = getRandomUniqueHexColorWithInterval(50);

    // Add a marker at the beginning of the path
    const startMarker = new mapboxgl.Marker().setLngLat(coords[0]).addTo(map);

    // Add a marker at the end of the path (initially hidden)
    endMarker = new mapboxgl.Marker({color: 'red'}).setLngLat(coords[coords.length - 1]).addTo(map);
    endMarker.getElement().style.display = 'none';

    // Add the path
    let path = {
      'type': 'geojson',
      'data': {
        'type': 'Feature',
        'properties': {
          'description': id,
        },
        'geometry': {
          'type': 'LineString',
          'coordinates': coords,
        },
      },
    };
    map.addSource(id, path);
    map.addLayer({
      'id': id,
      'type': 'line',
      'source': id,
      'layout': {
        'line-join': 'round',
        'line-cap': 'round',
      },
      'paint': {
        'line-color': pathColor,
        'line-width': 8,
        'line-opacity': 0, // Set initial opacity to 0
      },
    });

    // Set up pop-up and mouse events only for the start marker
    startMarker.setPopup(new mapboxgl.Popup().setHTML(id + '<br>' + `<button onclick="window.toggle3d('${id}')">Toggle 3D</button>`));
    startMarker.getElement().addEventListener('click', () => {
      togglePath(id, coords); // Pass coords to togglePath
      startMarker.togglePopup();
    });
  }

  function togglePath(id, coords) {
    // for toggling visibility of the path if different path is clicked
    if (currentlyVisiblePath !== null && currentlyVisiblePath !== id) {
      // Hide the currently visible path
      map.setPaintProperty(currentlyVisiblePath, 'line-opacity', 0);
    }

    const lineOpacity = map.getPaintProperty(id, 'line-opacity');
    const newOpacity = lineOpacity === 0 ? 1 : 0;

    // Toggle the opacity of the path
    map.setPaintProperty(id, 'line-opacity', newOpacity);

    // Update the coordinates of the end marker to the new path's end coordinates
    endMarker.setLngLat(coords[coords.length - 1]);

    // Toggle the visibility of the end marker
    if (newOpacity === 1) {
      endMarker.getElement().style.display = 'block';
      // addRandomMarkers(coords); // Add markers for the current path
    } else {
      endMarker.getElement().style.display = 'none';
    }

    // Fly to the beginning of the path when the button is clicked
    if (newOpacity === 1) {
      map.flyTo({
        center: coords[0],
        zoom: 16,
        essential: true,
        speed: 1.5,
        curve: 1.42,
        easing(t) {
          return t;
        },
      });
    }

    // Update the currently visible path
    currentlyVisiblePath = newOpacity === 1 ? id : null;
  }

  map.on('load', () => {
    // addPath(map, 'maps1', MAPS1DATA);
    // addPath(map, 'maps2', MAPS2DATA);
    // addPath(map, 'maps3', MAPS3DATA);
    addPath(map, 'run1', RUN1DATA);
    // addPath(map, 'run2', RUN2DATA);

    addPath(map, 'FLIGHT1DATA', FLIGHT1DATA);
    addPath(map, 'FLIGHT2DATA', FLIGHT2DATA);

    // addPath(map, 'wrun1', WRUN1DATA);
    // addPath(map, 'wrun2', WRUN2DATA);
    // add3DPath(map, 'wrun3', WRUN3DATA);
  });

  function calculateMeanValues(data) {
    if (data.length === 0) {
      return [0, 0]; // Return [0, 0] if the array is empty to avoid division by zero
    }

    // Initialize sums for both columns to zero
    let sumX = 0;
    let sumY = 0;

    // Iterate through the array and sum up the values in each column
    for (let i = 0; i < data.length; i++) {
      sumX += data[i][0];
      sumY += data[i][1];
    }

    // Calculate the mean values by dividing the sums by the number of data points
    const meanX = sumX / data.length;
    const meanY = sumY / data.length;

    return [meanX, meanY];
  }

  watch(
      () => stateKeys.map(key => state[key]),
      (newValues, oldValues) => {
        newValues.forEach((newValue, index) => {
          if (newValue !== oldValues[index]) {
            const checkFileExists = async (url) => {
              try {
                const response = await fetch(url, {
                  method: 'HEAD', // HEAD requests only get the headers, not the body
                  mode: 'no-cors'  // This is to avoid CORS issues
                });
                if (response.status === 404) {
                  console.log('File not found:', url);
                  return false;
                }
                return response.status === 200;
              } catch (error) {
                console.error('An error occurred while checking if the file exists:', error);
                return false;
              }
            };

            // file path for development
            // const filePath = `/3d/${stateKeys[index]}.glb`;

            // file path for deployment
            const filePath = `./3d/${stateKeys[index]}.glb`;
            checkFileExists(filePath).then(exists => {
              if (exists) {
                if (map.getLayer(`${stateKeys[index]}-3d`)) {
                  map.removeLayer(`${stateKeys[index]}-3d`);
                  return;
                }
                let stateKeyName = stateKeys[index];
                let dataSet = dataSets[stateKeyName];
                let offset = 0;
                if (stateKeyName === 'FLIGHT1DATA') {
                  offset = 30;
                }
                if (stateKeyName === 'FLIGHT2DATA') {
                  offset = 0;
                }
                map.addLayer(create3dModelLayer(dataSet[0], map.queryTerrainElevation(dataSet[0]) + offset, [Math.PI / 2, 0, 0], `${filePath}`, `${stateKeys[index]}-3d`), 'waterway-label');
              } else {
                console.log("There is no 3d model for this path.");
              }
            });
          }
        });
      },
      {deep: true}
  );

  watch(
      () => isTerrainEnabled.value,
      (newValue) => {
        if (newValue) {
          map.addSource('mapbox-dem', {
            'type': 'raster-dem',
            'url': 'mapbox://mapbox.mapbox-terrain-dem-v1',
            'tileSize': 512,
            'maxzoom': 14,
          });

          map.setTerrain({'source': 'mapbox-dem', 'exaggeration': 1.5});
        } else {
          map.setTerrain(null);
          map.removeSource('mapbox-dem');
        }
      }
  );

  watch(
      () => isSatelliteView.value,
      (newValue) => {
        if (newValue) {
          map.setStyle('mapbox://styles/theolchi/clnw50m2r003z01pg94do2adn');
        } else {
          map.setStyle('mapbox://styles/mapbox/outdoors-v11');
        }

        // Re-add paths and terrain after the style has been loaded
        map.once('style.load', () => {
          // addPath(map, 'maps1', MAPS1DATA);
          // addPath(map, 'maps2', MAPS2DATA);
          // addPath(map, 'maps3', MAPS3DATA);
          addPath(map, 'FLIGHT1DATA', FLIGHT1DATA);
          addPath(map, 'FLIGHT2DATA', FLIGHT2DATA);

          // Re-add terrain if it was enabled before
          if (isTerrainEnabled.value) {
            map.addSource('mapbox-dem', {
              'type': 'raster-dem',
              'url': 'mapbox://mapbox.mapbox-terrain-dem-v1',
              'tileSize': 512,
              'maxzoom': 14,
            });

            map.setTerrain({'source': 'mapbox-dem', 'exaggeration': 1.5});
          }
        });
      }
  );
});
</script>

<template>
  <div id="map"></div>
  <button @click="toggleTerrain()" id="toggleTerrainButton">Toggle Terrain</button>
  <br>
  <button @click="toggleMapView" id="toggleMapView">Toggle Map View</button>
</template>

<style scoped>
#toggleTerrainButton {
  position: relative;
  margin-top: 2%;
  left: 10px;
  z-index: 1;
}

#toggleMapView {
  position: relative;
  margin-top: 1%;
  left: 10px;
  z-index: 1;
}
</style>
