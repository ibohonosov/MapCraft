<template>
  <div class="flex flex-col items-center justify-center min-h-screen">
    <div
      ref="mapContainer"
      class="map-container border-2 border-blue-800 w-full rounded-md max-w-3xl h-100"
    ></div>
    <p class="text-center mt-6">
      Натисни на елемент (лінію, точку, полігон) та
    </p>
    <div class="form-container mt-3">
      <p class="text-center">Зміни колір елемента</p>
      <input
        type="color"
        v-model="color"
        @input="updateColor"
        class="input-color color-picker mx-auto block mt-2"
      />
      <p class="text-center mt-2">Зміни прозорість елемента</p>
      <input
        type="range"
        min="0"
        max="1"
        step="0.1"
        v-model="opacity"
        @input="updateOpacity"
        class="input-range opacity-slider mx-auto block mt-4 w-3/4"
      />
      <button
        @click="exportGeoJSON"
        class="btn-export bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded mt-6 mx-auto block"
      >
        Експорт GeoJSON
      </button>
    </div>
  </div>
</template>

<script>
import maplibregl from "maplibre-gl";
import * as turf from "@turf/turf";

export default {
  name: "MapComponent",
  data() {
    return {
      map: null,
      color: "#2236e2",
      opacity: 0.5,
      activeElementType: null,
      activeElementId: null,
    };
  },
  mounted() {
    this.initializeMap();
  },
  beforeUnmount() {
    if (this.map) {
      this.map.remove();
    }
  },
  methods: {
    initializeMap() {
      this.map = new maplibregl.Map({
        container: this.$refs.mapContainer,
        style: "https://demotiles.maplibre.org/style.json",
        center: [-0.1276, 51.5072],
        zoom: 5,
      });

      this.map.on("load", () => {
        const point = turf.point([-0.1276, 51.5072]);
        this.map.addSource("point", {
          type: "geojson",
          data: point,
        });
        this.map.addLayer({
          id: "point",
          type: "circle",
          source: "point",
          paint: {
            "circle-radius": 10,
            "circle-color": "#007cbf",
          },
        });
        const line = turf.lineString([
          [-0.1276, 51.5072], // Лондон
          [-84.006, 40.7128],
        ]);
        this.map.addSource("line", {
          type: "geojson",
          data: line,
        });
        this.map.addLayer({
          id: "line",
          type: "line",
          source: "line",
          layout: {
            "line-join": "round",
            "line-cap": "round",
          },
          paint: {
            "line-color": "#ff00ff",
            "line-width": 5,
          },
        });
        const polygon = turf.polygon([
          [
            [-0.1281, 51.5074], // Лондон
            [-74.0066, 40.7132], // Нью-Йорк
            [-3.7038, 40.4168], // Мадрид
            [-0.1281, 51.5074], // Лондон
          ],
        ]);
        this.map.addSource("polygon", {
          type: "geojson",
          data: polygon,
        });
        this.map.addLayer({
          id: "polygon",
          type: "fill",
          source: "polygon",
          layout: {},
          paint: {
            "fill-color": "#008000",
            "fill-opacity": 0.5,
          },
        });
      });
      this.map.on("click", "point", (e) =>
        this.selectElement("point", e.features[0].id)
      );
      this.map.on("click", "line", (e) =>
        this.selectElement("line", e.features[0].id)
      );
      this.map.on("click", "polygon", (e) =>
        this.selectElement("polygon", e.features[0].id)
      );
      ["point", "line", "polygon"].forEach((layerId) => {
        this.map.on("mouseenter", layerId, () => {
          this.map.getCanvas().style.cursor = "pointer";
        });
        this.map.on("mouseleave", layerId, () => {
          this.map.getCanvas().style.cursor = "";
        });
      });
    },

    selectElement(type, id) {
      this.activeElementType = type;
      this.activeElementId = id;
    },

    async exportGeoJSON() {
      await this.mapLoaded();

      this.performGeoJSONExport();
    },

    mapLoaded() {
      return new Promise((resolve) => {
        if (this.map && this.map.isStyleLoaded()) {
          resolve();
        } else {
          this.map.on("load", () => resolve());
        }
      });
    },

    updateColor() {
      if (!this.map || !this.map.isStyleLoaded() || !this.activeElementType)
        return;
      let paintProperty = "";
      if (this.activeElementType === "point") paintProperty = "circle-color";
      else if (this.activeElementType === "line") paintProperty = "line-color";
      else if (this.activeElementType === "polygon")
        paintProperty = "fill-color";

      this.map.setPaintProperty(
        this.activeElementType,
        paintProperty,
        this.color
      );
    },

    updateOpacity() {
      if (!this.map || !this.map.isStyleLoaded() || !this.activeElementType)
        return;

      const numericOpacity = parseFloat(this.opacity);
      let paintProperty = "";
      if (this.activeElementType === "point") paintProperty = "circle-opacity";
      else if (this.activeElementType === "line")
        paintProperty = "line-opacity";
      else if (this.activeElementType === "polygon")
        paintProperty = "fill-opacity";

      this.map.setPaintProperty(
        this.activeElementType,
        paintProperty,
        numericOpacity
      );
    },

    performGeoJSONExport() {
      if (!this.map) return;

      const data = {
        type: "FeatureCollection",
        features: [],
      };

      try {
        ["point", "line", "polygon"].forEach((id) => {
          data.features.push(this.map.getSource(id)._data);
        });
      } catch (error) {
        console.error("Помилка при експорті:", error);
        return;
      }

      const blob = new Blob([JSON.stringify(data)], {
        type: "application/json",
      });
      const url = URL.createObjectURL(blob);
      const link = document.createElement("a");
      link.href = url;
      link.download = "Geometry.geojson";
      document.body.appendChild(link);
      link.click();
      link.remove();
      URL.revokeObjectURL(url);
    },
  },
};
</script>
