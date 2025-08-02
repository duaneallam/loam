<script setup lang="ts">
import { defineProps, onMounted } from 'vue';
import type { FeatureCollection, Polygon, Position } from 'geojson';
import maplibregl from 'maplibre-gl';
import 'maplibre-gl/dist/maplibre-gl.css';

const colours = [
  '#E6C229', // Yellow
  '#F17105', // Orange
  '#D11149', // Red
  '#6610F2', // Purple
  '#1A8FE3'  // Blue
];

const props = defineProps<{
  geoJSON: FeatureCollection;
}>();

const project_ids = Array.from(
  new Set(
    props.geoJSON.features.map(feature => feature.properties?.Project__Name)
      .filter((v) => v !== null)
  )
);

const project_colours = Object.fromEntries(
  project_ids.map((project, index) => [project, colours[index % colours.length]])
);

const project_summary =
  project_ids.map(project_id => {

    const project_features = props.geoJSON.features.filter(feature => feature.properties?.Project__Name === project_id);

    const totalArea = project_features.reduce((sum, feature) => {
      return sum + (feature.properties?.area_acres || 0);
    }, 0);

    return {
      name: project_id,
      colour: project_colours[project_id],
      count: project_features.length,
      totalArea: totalArea,
      owner: project_features[0]?.properties?.owner,
    };
  });

const processedFeatures = processFeatures(props.geoJSON, project_colours);

function findCentre(coordinates: Position[]): Position {
  const bounds = new maplibregl.LngLatBounds();
  coordinates.forEach(coord => {
    bounds.extend(coord as [number, number]);
  });
  return bounds.getCenter().toArray() as Position;
}


function processFeatures(features: FeatureCollection, project_colours: { [project_name: string]: string }): FeatureCollection {

  return {
    type: 'FeatureCollection',
    features: features.features.map(feature => {
      let colour = '#888888';
      if (feature.properties?.Project__Name) {
        colour = project_colours[feature.properties.Project__Name];
      }
      const description = `<strong>${feature.properties?.name}</strong>` +
        `<p>Owner: ${feature.properties?.owner}` +
        `<br/>Area: ${feature.properties?.area_acres} acres` +
        (feature.properties?.Project__Name ? `<br/>Project: ${feature.properties.Project__Name}` : '') +
        `</p>`;

      return {
        ...feature,
        properties: {
          ...feature.properties,
          fillColor: colour,
          description: description
        }
      };
    })
  };
}
onMounted(() => {
  const coordinates = processedFeatures.features.flatMap(feature => {
    return (feature.geometry as Polygon).coordinates[0];
  });

  const map = new maplibregl.Map({
    container: 'map', // container id
    style: 'https://api.maptiler.com/maps/hybrid/style.json?key=ptCw7SIlRyyxWPexmhu9', // style URL
    center: findCentre(coordinates) as [number, number], // starting position [lng, lat]
    zoom: 11.5 // starting zoom
  });

  map.on('load', () => {

    map.addSource('paddocks', {
      type: 'geojson',
      data: processedFeatures
    });

    map.addLayer({
      id: 'paddock-layer',
      type: 'fill',
      source: 'paddocks',
      layout: {},
      paint: {
        'fill-color': ['get', 'fillColor'],
        'fill-opacity': 0.5
      }
    });

    const popup = new maplibregl.Popup({
      closeButton: true,
      className: 'map-popup'
    });



    // Make sure to detect marker change for overlapping markers
    // and use mousemove instead of mouseenter event
    let currentFeatureCoordinates = "";

    map.on('click', 'paddock-layer', (e) => {
      if (e.features) {

        const geometry = e.features[0].geometry as Polygon;
        const featureCoordinates = geometry.coordinates.toString();
        if (currentFeatureCoordinates !== featureCoordinates) {
          currentFeatureCoordinates = featureCoordinates;

          const description = e.features[0].properties.description;

          // Populate the popup and set its coordinates
          // based on the feature found.
          popup.setLngLat(findCentre(geometry.coordinates[0]) as [number, number]).setHTML(description).addTo(map);
        }
      }
    });

    popup.on('close', () => {
      currentFeatureCoordinates = "";
    });

    map.on('mouseleave', 'paddock-layer', () => {
      map.getCanvas().style.cursor = '';
    });
    map.on('mouseenter', 'paddock-layer', () => {
      map.getCanvas().style.cursor = 'pointer';
    });

  });
})

</script>

<template>
  <div id="map"></div>
  <h2>Project Summary</h2>
  <div class="project-summary">
    <ul>
      <li v-for="(project, index) in project_summary" :key="index">
        <span class="color-box" :style="{ backgroundColor: project.colour }"></span>
        <span class="name">{{ project.name }}</span>
        <span class="area">total {{ project.totalArea }} acres</span>
        <span class="count">{{ project.count }} paddocks</span>
        <span class="owner">Owner: {{ project.owner }}</span>
      </li>
    </ul>
  </div>
</template>

<style scoped></style>
