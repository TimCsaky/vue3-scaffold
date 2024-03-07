<script setup lang="ts">
import * as L from 'leaflet';

import 'leaflet/dist/leaflet.css';
import '@geoman-io/leaflet-geoman-free';
import '@geoman-io/leaflet-geoman-free/dist/leaflet-geoman.css';

import { onMounted, ref, watch } from 'vue';
import { Button, Column, DataTable, Dropdown, InputText } from '@/lib/primevue';

import { dataService } from '@/services';
import type { Geometry } from 'geojson';
import type { Ref } from 'vue';

const layers: Ref<Array<{ name: string; layer: L.GeoJSON<any, Geometry> }>> = ref([]);
const selectedLayer: Ref<L.GeoJSON<any, Geometry> | undefined> = ref(undefined);
const newLayerName: Ref<string> = ref('');

let map: L.Map;
let layerControl: L.Control.Layers;

const parcelData = ref('a');

// Actions
function createLayer() {
  if (newLayerName.value && newLayerName.value.length > 0) {
    // add layer to map
    const newLayer = L.geoJSON().addTo(map);
    layerControl.addOverlay(newLayer, newLayerName.value);
    // add to local state
    layers.value.push({ name: newLayerName.value, layer: newLayer });
    // set as selected layer
    selectedLayer.value = newLayer;

    // on polygon create
    map.on('pm:create', async (e) => {
      // zoom in
      map.fitBounds(e.layer.getBounds());
      // show parcel data
      showParcelData(e.layer.getLatLngs()[0]);
    });
  }
}

async function showParcelData(data) {
  await dataService.getParcelData(data).then((data) => {
    parcelData.value = data.features.map((f) => f.properties);
  });
}

function deleteLayer() {
  if (selectedLayer.value) {
    selectedLayer.value.removeFrom(map);
    layerControl.removeLayer(selectedLayer.value);
    layers.value = layers.value.filter((x) => x.layer !== selectedLayer.value);
    selectedLayer.value = layers.value[0].layer;
  }
}

watch(selectedLayer, () => {
  if (selectedLayer.value) {
    map.pm.setGlobalOptions({ layerGroup: selectedLayer.value });
  }
});

onMounted(() => {
  const osm = L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
    maxZoom: 19,
    attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>'
  });

  map = L.map('map', {
    center: [48.428, -123.365],
    zoom: 13,
    layers: [osm]
  });

  // add Leaflet-Geoman controls with some options to the map
  map.pm.addControls({
    position: 'topleft',
    drawCircleMarker: false,
    rotateMode: false
  });

  const baseMaps = {
    OpenStreetMap: osm
  };

  layers.value = [];

  const overlayMaps = {};

  layerControl = L.control.layers(baseMaps, overlayMaps).addTo(map);

  // selectedLayer.value = geoLayer1;
});
</script>

<template>
  <div class="flex flex-row w-full">
    <div class="flex flex-column mr-2">
      <p class="font-bold mt-0 mb-0">Select draw layer</p>
      <Dropdown
        v-model="selectedLayer"
        class="mb-2"
        :options="layers"
        option-label="name"
        option-value="layer"
      />
      <Button
        severity="danger"
        @click="deleteLayer"
      >
        Delete layer
      </Button>
      <p class="font-bold mt-2 mb-0">Create layer</p>
      <div class="flex">
        <InputText
          v-model="newLayerName"
          class="mr-1"
        />
        <Button
          icon="pi pi-check"
          @click="createLayer"
        />
      </div>
      <!-- parcel data -->
      <!-- <ul v-if="parcelData.length">
        <li v-for="parcelId of parcelData">{{ parcelId }}</li>
      </ul> -->
    </div>
    <div
      id="map"
      class="flex flex-auto"
    />
  </div>

  <div
    v-if="parcelData.length"
    class="py-2 mw-50"
  >
    <h3>Parcel Data</h3>
    <DataTable
      :value="parcelData"
      data-key="PID"
      class="p-datatable-sm"
      responsive-layout="scroll"
      :paginator="true"
      :rows="5"
    >
      <Column
        field="PID"
        header="Parcel ID"
      ></Column>
      <Column
        field="OWNER_TYPE"
        header="Owner Type"
      ></Column>
      <Column
        field="PARCEL_CLASS"
        header="Parcel Class"
      ></Column>
    </DataTable>
  </div>
</template>

<style scoped lang="scss">
#map {
  height: 500px;
}
</style>
