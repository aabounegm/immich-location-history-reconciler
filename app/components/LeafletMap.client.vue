<script setup lang="ts">
import { onMounted, ref } from "vue";
import L from "leaflet";
import markerIconSrc from "leaflet/dist/images/marker-icon.png";
import markerIconRetinaSrc from "leaflet/dist/images/marker-icon-2x.png";
import markerShadowSrc from "leaflet/dist/images/marker-shadow.png";
import "leaflet/dist/leaflet.css";
import { GeoSearchControl, OpenStreetMapProvider } from "leaflet-geosearch";
import "leaflet-geosearch/assets/css/leaflet.css";
import type { LatLng } from "~/lib/timeline";
import { type Geometry } from "~/lib/geometry";

if (!import.meta.dev) {
  L.Icon.Default.mergeOptions({
    iconUrl: markerIconSrc,
    iconRetinaUrl: markerIconRetinaSrc,
    shadowUrl: markerShadowSrc,
  });
}

const bestLocation = defineModel<LatLng>();

const { segments = [] } = defineProps<{
  segments?: Geometry[];
}>();

const mapContainer = templateRef("leafletMap");

const map = ref<L.Map>();
let bestLocationMarker: L.Marker | undefined;
const sourceSegmentsLayer = L.layerGroup();

const provider = new OpenStreetMapProvider();
const searchControl = GeoSearchControl({
  provider: provider,
  showMarker: false,
});

onMounted(() => {
  map.value = L.map(mapContainer.value!).setView([0, 0], 0);
  map.value.attributionControl.setPrefix("Leaflet");
  map.value.addControl(searchControl);

  L.tileLayer("https://tile.openstreetmap.org/{z}/{x}/{y}.png", {
    maxZoom: 19,
    attribution:
      '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>',
  }).addTo(map.value);

  map.value.addLayer(sourceSegmentsLayer);

  map.value.on("click", (e) => {
    bestLocation.value = e.latlng;
  });
});

const redMarkerIcon = new L.Icon.Default({
  className: "hue-rotate-[120deg]",
});

watch(
  [bestLocation, map],
  ([location, map]) => {
    if (!map) return;

    if (location) {
      if (bestLocationMarker) {
        bestLocationMarker.setLatLng(location);
      } else {
        map.setView(location, 15);
        bestLocationMarker = L.marker(location, {
          icon: redMarkerIcon,
          draggable: true,
          autoPan: true,
        }).addTo(map);
        bestLocationMarker.on("dragend", () => {
          if (!bestLocationMarker) return;
          const latLng = bestLocationMarker.getLatLng();
          bestLocationMarker.setLatLng(latLng);
        });
      }
    } else if (bestLocationMarker) {
      bestLocationMarker.remove();
      bestLocationMarker = undefined;
    }
  },
  { immediate: true }
);

watch(
  [() => segments, map],
  ([segments, map]) => {
    if (!map) return;

    sourceSegmentsLayer.clearLayers();
    const bounds: L.LatLngTuple[] = [];

    segments.forEach((segment) => {
      switch (segment.type) {
        case "path": {
          const line = L.polyline(
            segment.points.map(({ point }) => point),
            {
              color: "red",
              weight: 3,
              opacity: 0.8,
              interactive: false,
            }
          );
          sourceSegmentsLayer.addLayer(line);

          const circles = segment.points.map(({ point, time }) =>
            L.circleMarker(point, {
              color: "red",
              fillColor: "red",
              fillOpacity: 0.5,
              radius: 3,
            }).bindPopup(time.toLocaleString())
          );
          circles.forEach((circle) => sourceSegmentsLayer.addLayer(circle));

          bounds.push(
            ...segment.points.map(
              ({ point: { lat, lng } }) => [lat, lng] satisfies L.LatLngTuple
            )
          );

          break;
        }
        case "activity": {
          const { start, end, activityType } = segment;
          const line = L.polyline([start.point, end.point], {
            color: "blue",
            weight: 3,
            opacity: 0.8,
          }).bindPopup(`Activity: ${activityType}`, { maxWidth: 200 });
          sourceSegmentsLayer.addLayer(line);

          const startCircle = L.circleMarker(start.point, {
            color: "blue",
            fillColor: "blue",
            fillOpacity: 0.5,
            radius: 3,
          }).bindPopup(`Start: ${start.time.toLocaleString()}`);
          const endCircle = L.circleMarker(end.point, {
            color: "blue",
            fillColor: "blue",
            fillOpacity: 0.5,
            radius: 3,
          }).bindPopup(`End: ${end.time.toLocaleString()}`);

          sourceSegmentsLayer.addLayer(startCircle);
          sourceSegmentsLayer.addLayer(endCircle);

          bounds.push(
            [start.point.lat, start.point.lng],
            [end.point.lat, end.point.lng]
          );

          break;
        }
        case "visit": {
          const { point, timeRange } = segment;
          const marker = L.marker(point).bindPopup(
            `Visit: ${timeRange.start.toLocaleString()} - ${timeRange.end.toLocaleString()}`,
            { maxWidth: 200 }
          );
          sourceSegmentsLayer.addLayer(marker);

          bounds.push([point.lat, point.lng]);

          break;
        }
      }
    });

    if (bounds.length) {
      map.fitBounds(bounds);
    }
  },
  { immediate: true }
);
</script>

<template>
  <div class="min-h-60" ref="leafletMap" />
</template>
