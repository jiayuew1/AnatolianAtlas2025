<template>
  <div id="map" class="map-container"></div>
</template>

<script>
import { toRaw } from 'vue';
import "leaflet/dist/leaflet.css";
import "leaflet.markercluster/dist/MarkerCluster.css";
import "leaflet.markercluster/dist/MarkerCluster.Default.css";
import L from "leaflet";
import "leaflet.markercluster";

export default {
  name: "ClusterMap",
  
  props: {
    points: {
      type: Array,
      required: true
    },
    getColor: {
      type: Function,
      required: true
    }
  },
  
  data() {
    return {
      map: null,
      markerClusterGroup: null,
      tileLayer: null
    };
  },
  
  mounted() {
    this.$nextTick(() => {
      this.initializeMap();
      this.initializeMarkers();
    });
  },
  
  methods: {
    initializeMap() {
      // Create map with Vue 3 toRaw to prevent proxying issues
      this.map = L.map("map", {
        center: [37.5, 33.5],
        zoom: 7
      });
      
      // Add tile layer from OpenStreetMap
      this.tileLayer = L.tileLayer(
        "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",
        {
          attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
        }
      ).addTo(toRaw(this.map));

      // Initialize marker cluster group using toRaw
      this.markerClusterGroup = L.markerClusterGroup({
        chunkedLoading: true,
        maxClusterRadius: 50,
        spiderfyOnMaxZoom: true,
        showCoverageOnHover: true,
        zoomToBoundsOnClick: true,
        disableClusteringAtZoom: 10,
        iconCreateFunction: this.createClusterIcon
      });

      // Add marker cluster group to the map using toRaw
      toRaw(this.map).addLayer(toRaw(this.markerClusterGroup));
    },
    
    createClusterIcon(cluster) {
      // Calculate color distribution in the cluster
      const markers = cluster.getAllChildMarkers();
      const colorCounts = {};
      
      markers.forEach(marker => {
        const color = marker.options.fillColor;
        colorCounts[color] = (colorCounts[color] || 0) + 1;
      });
      
      // Determine dominant color
      const dominantColor = Object.entries(colorCounts).length > 0
        ? Object.entries(colorCounts).reduce((a, b) => a[1] > b[1] ? a : b)[0]
        : '#999999';
      
      // Determine cluster size
      const count = cluster.getChildCount();
      let size = 'small';
      if (count > 100) {
        size = 'large';
      } else if (count > 10) {
        size = 'medium';
      }
      
      // Create and return custom div icon
      return L.divIcon({
        html: `<div style="background-color: ${dominantColor}"><span>${count}</span></div>`,
        className: `marker-cluster marker-cluster-${size}`,
        iconSize: L.point(40, 40)
      });
    },
    
    initializeMarkers() {
      // Use toRaw to prevent proxying issues
      const rawMarkerClusterGroup = toRaw(this.markerClusterGroup);
      if (!rawMarkerClusterGroup) return;
      
      // Clear existing layers
      rawMarkerClusterGroup.clearLayers();
      
      // Create markers for each point
      const markers = this.points.map(point => {
        const marker = L.circleMarker(
          [parseFloat(point.y), parseFloat(point.x)],
          {
            radius: 6,
            fillColor: this.getColor(point.Survey),
            color: this.getColor(point.Survey),
            weight: 1,
            opacity: 1,
            fillOpacity: 0.8
          }
        ).bindPopup(`
          <div class="popup-content">
            <h3>${point['Name']}</h3>
            <p><strong>Survey:</strong> ${point.Survey}</p>
            <p><strong>Periods:</strong> ${point.News ? point.News.split(',').join(', ') : ''}</p>
          </div>
        `, {
          maxWidth: 300,
          maxHeight: 250,
          autoPan: true,
          closeButton: true,
          className: 'custom-popup'
        });
        
        return marker;
      });
      
      // Add markers in batch using toRaw
      rawMarkerClusterGroup.addLayers(markers);
    }
  },
  
  // Watch for changes in points
  watch: {
    points: {
      handler() {
        this.$nextTick(() => {
          this.initializeMarkers();
        });
      },
      deep: true
    }
  },
  
  // Cleanup before component is unmounted
  beforeUnmount() {
    if (this.map) {
      // Use toRaw for map removal
      toRaw(this.map).remove();
    }
  }
};
</script>

<style>
.map-container {
  height: 100%;
  width: 100%;
}

/* Cluster marker styling */
.marker-cluster {
  background-clip: padding-box;
  border-radius: 20px;
}

.marker-cluster div {
  width: 30px;
  height: 30px;
  margin-left: 5px;
  margin-top: 5px;
  text-align: center;
  border-radius: 15px;
  font: 12px "Helvetica Neue", Arial, Helvetica, sans-serif;
  display: flex;
  align-items: center;
  justify-content: center;
}

.marker-cluster span {
  color: white;
}

/* Cluster size variations */
.marker-cluster-small {
  width: 40px;
  height: 40px;
}

.marker-cluster-medium {
  width: 50px;
  height: 50px;
}

.marker-cluster-large {
  width: 60px;
  height: 60px;
}

/* Popup styling */
:deep(.leaflet-popup-content-wrapper) {
  max-height: 250px;
  overflow-y: auto;
  padding: 8px;
}

:deep(.leaflet-popup-content) {
  margin: 8px;
  width: auto !important;
}

:deep(.custom-popup .leaflet-popup-content-wrapper) {
  border-radius: 8px;
  padding: 12px;
  box-shadow: 0 3px 14px rgba(0,0,0,0.2);
}

:deep(.custom-popup .leaflet-popup-content p) {
  margin: 4px 0;
}
</style>