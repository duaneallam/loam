<script setup lang="ts">
import PaddockMap from './components/PaddockMap.vue';

import paddockJSONRaw from './assets/sample_paddocks.json';
import type { FeatureCollection } from 'geojson';

const paddockJSON = paddockJSONRaw as FeatureCollection;


</script>

<template>
  <h1>Loam coding challenge</h1>
  <PaddockMap :geoJSON="paddockJSON" />
</template>

<style scoped>
.logo {
  height: 6em;
  padding: 1.5em;
  will-change: filter;
  transition: filter 300ms;
}

.logo:hover {
  filter: drop-shadow(0 0 2em #646cffaa);
}

.logo.vue:hover {
  filter: drop-shadow(0 0 2em #42b883aa);
}
</style>
