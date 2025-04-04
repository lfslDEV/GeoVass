<template>
  <div class="property-map">
    <div class="map-header">
      <div class="brand">
        <div class="logo">
          <i class="fas fa-globe-americas"></i>
          <h2>GeoVass</h2>
        </div>
        <span class="subtitle">Sistema de Medição de Terrenos</span>
      </div>

      <div class="toolbar">
        <div class="search-bar">
          <label for="address-search" class="sr-only">Buscar endereço</label>
          <i class="fas fa-search" aria-hidden="true"></i>
          <input 
            id="address-search"
            type="search"
            v-model="searchQuery" 
            placeholder="Buscar endereço..."
            @keyup.enter="searchLocation"
            @input="handleSearchInput"
            autocomplete="off"
            aria-label="Buscar endereço"
          >
          <div v-if="isSearching" class="search-loading">
            <i class="fas fa-circle-notch fa-spin"></i>
          </div>
        </div>
        <div class="tools">
          <button class="tool-button" @click="toggleHistory" title="Histórico">
            <i class="fas fa-history"></i>
          </button>
        </div>
      </div>

      <div class="instructions-panel">
        <div class="instructions-header">
          <i class="fas fa-info-circle"></i>
          <span>Guia Rápido</span>
        </div>
        <div class="instructions-content">
          <div class="instruction-step" v-for="(step, index) in instructions" :key="index">
            <span class="step-number">{{ index + 1 }}</span>
            <p v-html="step"></p>
          </div>
        </div>
      </div>
    </div>

    <div class="map-wrapper">
      <div class="measurement-history" v-if="showHistory">
        <div class="history-header">
          <h3>Histórico de Medições</h3>
          <button class="close-button" @click="toggleHistory">×</button>
        </div>
        <div class="history-content">
          <div v-for="(item, index) in measurementHistory" 
               :key="index" 
               class="history-item"
               @click="loadMeasurement(item)">
            <div class="history-item-header">
              <span class="history-date">{{ formatDate(item.date) }}</span>
            </div>
            <div class="history-values">
              <div class="history-value">
                <i class="fas fa-vector-square"></i>
                Área: {{ formatArea(item.area) }}
              </div>
              <div class="history-value">
                <i class="fas fa-ruler"></i>
                Perímetro: {{ formatPerimeter(item.perimeter) }}
              </div>
            </div>
          </div>
        </div>
      </div>

      <div id="map" class="map-container" ref="mapContainer">
        <div v-if="isLoading" class="loading-overlay">
          <div class="spinner"></div>
          <p>Inicializando GeoMensor Pro...</p>
        </div>

        <!-- Enhanced Measurement Display -->
        <div v-if="currentMeasurement" class="info-panel">
          <div class="info-section measurement-section">
            <div class="section-header">
              <i class="fas fa-ruler-combined"></i>
              <h3>Medições Atuais</h3>
            </div>
            <div class="measurement-grid">
              <div class="measurement-item">
                <div class="measurement-label">
                  <i class="fas fa-vector-square"></i>
                  <span>Área</span>
                </div>
                <div class="measurement-value">
                  <input 
                    type="text" 
                    readonly
                    :value="formatArea(currentMeasurement.area)"
                    class="selectable-value"
                    @click="$event.target.select()"
                  >
                  <button 
                    class="copy-button" 
                    @click="copyToClipboard(formatArea(currentMeasurement.area))"
                    title="Copiar área"
                  >
                    <i class="fas fa-copy"></i>
                  </button>
                </div>
              </div>
              <div class="measurement-item">
                <div class="measurement-label">
                  <i class="fas fa-ruler"></i>
                  <span>Perímetro</span>
                </div>
                <div class="measurement-value">
                  <input 
                    type="text" 
                    readonly
                    :value="formatPerimeter(currentMeasurement.perimeter)"
                    class="selectable-value"
                    @click="$event.target.select()"
                  >
                  <button 
                    class="copy-button" 
                    @click="copyToClipboard(formatPerimeter(currentMeasurement.perimeter))"
                    title="Copiar perímetro"
                  >
                    <i class="fas fa-copy"></i>
                  </button>
                </div>
              </div>
            </div>
            <div class="measurement-actions">
              <button @click="saveMeasurement" title="Salvar medição" class="action-button">
                <i class="fas fa-save"></i>
                <span>Salvar</span>
              </button>
            </div>
          </div>

          <div class="info-section coordinates-section">
            <div class="section-header">
              <i class="fas fa-map-marker-alt"></i>
              <h3>Coordenadas</h3>
            </div>
            <div class="coordinates-list">
              <div v-for="(point, index) in currentPolygonPoints" :key="index" class="coordinate-item">
                <span class="point-number">{{ index + 1 }}</span>
                <span class="coordinate">{{ formatCoordinate(point) }}</span>
              </div>
            </div>
          </div>
        </div>
        <div class="map-overlay-controls">
          <!-- Remove old layer controls -->
        </div>

        <!-- Add new layer selector in bottom right -->
        <div class="map-layer-selector">
          <div class="layer-controls">
            <button 
              v-for="layer in availableLayers" 
              :key="layer.id"
              class="layer-button"
              :class="{ active: currentLayer === layer.id }"
              @click="currentLayer = layer.id"
              :title="layer.name"
            >
              <i :class="layer.icon"></i>
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>

  <div class="signature">
    <p>Desenvolvido por Luis Filipe, Duda e Kayo</p>
    <p>Secretaria de Desenvolvimento Econômico e Inovação</p>
  </div>
</template>

<script>
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';
import 'leaflet-draw/dist/leaflet.draw.css';
import 'leaflet-draw';
import * as turf from '@turf/turf';

// Constants for better maintainability and performance
const MAP_CONFIG = {
  CENTER: [-22.4039, -43.6628],
  ZOOM: {
    DEFAULT: 19,
    MAX: 20,
    MIN: 10,
    PRECISION: 20
  },
  TILE_URL: 'https://{s}.google.com/vt/lyrs=s,h&x={x}&y={y}&z={z}&hl=pt-BR',
  SUBDOMAINS: ['mt0', 'mt1', 'mt2', 'mt3']
};

const MEASUREMENT_UNITS = {
  area: [
    { value: 'meters', label: 'm²', conversion: 1 },
    { value: 'hectares', label: 'ha', conversion: 0.0001 },
    { value: 'feet', label: 'ft²', conversion: 10.764 }
  ],
  perimeter: [
    { value: 'meters', label: 'm', conversion: 1 },
    { value: 'kilometers', label: 'km', conversion: 0.001 },
    { value: 'feet', label: 'ft', conversion: 3.28084 }
  ]
};

const AVAILABLE_LAYERS = [
  { 
    id: 'hybrid', 
    name: 'Híbrido', 
    icon: 'fas fa-layer-group',
    url: 'https://{s}.google.com/vt/lyrs=s,h&x={x}&y={y}&z={z}&hl=pt-BR'
  },
  { 
    id: 'terrain', 
    name: 'Terreno', 
    icon: 'fas fa-mountain',
    url: 'https://{s}.google.com/vt/lyrs=p&x={x}&y={y}&z={z}&hl=pt-BR'
  }
];

const INSTRUCTIONS = [
  'Clique na ferramenta polígono <i class="fas fa-draw-polygon"></i> no canto superior direito',
  'Clique nos pontos para demarcar o perímetro do terreno',
  'Faça duplo clique para finalizar a medição',
  'Use os controles abaixo para visualizar as medições'
];

export default {
  name: 'PropertyMap',
  data() {
    return {
      map: null,
      drawnItems: null,
      tileLayer: null,
      drawControl: null,
      isLoading: true,
      currentMeasurement: null,
      currentLayer: 'hybrid',
      selectedUnit: 'meters',
      measurementHistory: [],
      showHistory: false,
      isPrecisionMode: false,
      gridLayer: null,
      instructions: INSTRUCTIONS,
      currentPolygonPoints: [],
      measurementUnits: MEASUREMENT_UNITS,
      searchQuery: '',
      isSearching: false,
      searchTimeout: null,
      searchDebounceTime: 500,
      searchMinLength: 3
    };
  },
  computed: {
    hasDrawnItems() {
      return this.drawnItems?.getLayers().length > 0;
    },
    availableLayers() {
      return AVAILABLE_LAYERS;
    },
    formattedMeasurements() {
      if (!this.currentMeasurement) return { area: '', perimeter: '' };
      return {
        area: this.formatArea(this.currentMeasurement.area),
        perimeter: this.formatPerimeter(this.currentMeasurement.perimeter)
      };
    }
  },
  watch: {
    currentLayer: {
      handler(newLayer) {
        this.updateMapLayer(newLayer);
      }
    }
  },
  mounted() {
    this.initMap();
    this.setupKeyboardShortcuts();
    this.loadSavedMeasurements();
  },
  beforeDestroy() {
    this.cleanup();
  },
  methods: {
    async initMap() {
      try {
        const mapContainer = this.$refs.mapContainer;
        if (!mapContainer) {
          throw new Error('Container do mapa não encontrado');
        }

        await this.initializeMap(mapContainer);
        await this.setupMapLayers();
        this.setupEventListeners();
        
        await this.$nextTick();
        this.map.invalidateSize();
        this.isLoading = false;
      } catch (error) {
        console.error('Erro ao inicializar o mapa:', error);
        this.handleError(error);
      }
    },

    initializeMap(container) {
      this.map = L.map(container, {
        center: MAP_CONFIG.CENTER,
        zoom: MAP_CONFIG.ZOOM.DEFAULT,
        maxZoom: MAP_CONFIG.ZOOM.MAX,
        minZoom: MAP_CONFIG.ZOOM.MIN,
        zoomControl: false,
        preferCanvas: true, // Better performance for vector layers
        wheelDebounceTime: 100,
        wheelPxPerZoomLevel: 100,
        fadeAnimation: true,
        zoomAnimation: true
      });
    },

    async setupMapLayers() {
      await this.addTileLayer();
      this.addGridLayer();
      this.setupDrawLayer();
    },

    async addTileLayer() {
      if (this.tileLayer) {
        this.map.removeLayer(this.tileLayer);
      }

      const currentLayerConfig = AVAILABLE_LAYERS.find(l => l.id === this.currentLayer) || AVAILABLE_LAYERS[0];

      this.tileLayer = L.tileLayer(currentLayerConfig.url, {
        subdomains: MAP_CONFIG.SUBDOMAINS,
        maxZoom: MAP_CONFIG.ZOOM.MAX,
        maxNativeZoom: MAP_CONFIG.ZOOM.MAX,
        attribution: 'Google Maps',
        keepBuffer: 4,
        updateWhenIdle: true,
        updateWhenZooming: false
      });

      this.tileLayer.on('tileerror', this.handleTileError);
      this.tileLayer.on('load', this.handleTileLoad);

      await new Promise(resolve => {
        this.tileLayer.on('load', resolve);
        this.tileLayer.addTo(this.map);
      });
    },

    handleTileError(error) {
      console.warn('Erro ao carregar tile:', error);
    },

    handleTileLoad() {
      // Handle successful tile load if needed
    },

    addGridLayer() {
      this.gridLayer = L.gridLayer({
        tileSize: 256,
        opacity: 0.4,
        zIndex: 1,
        keepBuffer: 2
      });
      this.gridLayer.addTo(this.map);
    },

    setupDrawLayer() {
      this.drawnItems = new L.FeatureGroup();
      this.map.addLayer(this.drawnItems);
      this.initializeDrawControl();
    },

    setupEventListeners() {
      // Map events
      this.map.on(L.Draw.Event.CREATED, this.handleDrawCreated);
      this.map.on(L.Draw.Event.EDITED, this.handleDrawEdited);
      this.map.on(L.Draw.Event.DELETED, this.handleDrawDeleted);
      this.map.on('moveend', this.debounce(this.handleMapMoveEnd, 250));
      this.map.on('zoomend', this.debounce(this.handleZoomEnd, 250));

      // Window events
      window.addEventListener('resize', this.debounce(this.handleResize, 250));
    },

    handleDrawCreated(event) {
      const layer = event.layer;
      this.drawnItems.addLayer(layer);
      this.updateMeasurement(layer);
    },

    handleDrawEdited(event) {
      event.layers.eachLayer(layer => this.updateMeasurement(layer));
    },

    handleDrawDeleted() {
      this.currentMeasurement = null;
      this.currentPolygonPoints = [];
    },

    handleMapMoveEnd() {
      // Handle map move end if needed
    },

    handleZoomEnd() {
      // Handle zoom end if needed
    },

    handleResize() {
      if (this.map) {
        this.map.invalidateSize();
      }
    },

    debounce(func, wait) {
      let timeout;
      return (...args) => {
        clearTimeout(timeout);
        timeout = setTimeout(() => func.apply(this, args), wait);
      };
    },

    cleanup() {
      if (this.map) {
        this.map.remove();
      }
      window.removeEventListener('resize', this.debounce(this.handleResize, 250));
    },

    initializeDrawControl() {
      this.drawControl = new L.Control.Draw({
        position: 'topright',
        draw: {
          polygon: {
            allowIntersection: false,
            drawError: {
              color: '#FF5252',
              message: '<strong>Erro!</strong> Polígonos não podem se intersectar!'
            },
            shapeOptions: {
              color: '#2196F3',
              fillColor: '#2196F3',
              fillOpacity: 0.15,
              weight: 2,
              opacity: 0.9,
              dashArray: null,
              lineCap: 'round',
              lineJoin: 'round'
            },
            showArea: true,
            metric: true,
            guidelineDistance: 5,
            precisionMode: true
          },
          polyline: false,
          circle: false,
          circlemarker: false,
          rectangle: false,
          marker: false
        },
        edit: {
          featureGroup: this.drawnItems,
          remove: true,
          edit: {
            selectedPathOptions: {
              color: '#1976D2',
              fillColor: '#2196F3',
              fillOpacity: 0.2,
              weight: 2,
              opacity: 0.8,
              dashArray: null
            },
            moveMarkers: true,
            snapDistance: 8,
            snapVertices: true
          }
        }
      });

      const style = document.createElement('style');
      style.textContent = `
        .leaflet-draw-tooltip {
          background: #2196F3;
          border: none;
          color: white;
          font-weight: 500;
          border-radius: 4px;
          padding: 8px 12px;
          box-shadow: 0 2px 8px rgba(33, 150, 243, 0.2);
        }
        .leaflet-draw-tooltip:before {
          border-right-color: #2196F3;
        }
        .leaflet-draw-tooltip-single {
          display: none;
        }
        .leaflet-draw-tooltip-subtext {
          color: rgba(255, 255, 255, 0.8);
          font-size: 0.9em;
          margin-top: 4px;
        }
        .leaflet-edit-marker-selected {
          background: #2196F3;
          border: 2px solid white;
          border-radius: 50%;
          box-shadow: 0 0 0 2px rgba(33, 150, 243, 0.5);
        }
        .leaflet-edit-marker-icon {
          background: #2196F3;
          border: 2px solid white;
          border-radius: 50%;
          box-shadow: 0 0 0 2px rgba(33, 150, 243, 0.3);
          width: 10px !important;
          height: 10px !important;
          margin-left: -5px !important;
          margin-top: -5px !important;
          transition: all 0.3s ease;
        }
        .leaflet-edit-marker-icon:hover {
          background: #1976D2;
          transform: scale(1.3);
          box-shadow: 0 0 0 3px rgba(33, 150, 243, 0.5);
        }
        .leaflet-edit-marker-icon:active {
          background: #1565C0;
          transform: scale(0.9);
        }
        .leaflet-draw-actions {
          background: white;
          border-radius: 4px;
          box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
          overflow: hidden;
        }
        .leaflet-draw-actions a {
          background: white;
          color: #2196F3;
          font-weight: 500;
          transition: all 0.2s;
        }
        .leaflet-draw-actions a:hover {
          background: #E3F2FD;
          color: #1976D2;
        }
      `;
      document.head.appendChild(style);

      this.map.addControl(this.drawControl);
    },

    updateMeasurement(layer) {
      if (!layer) return;
      
      try {
        const geoJson = layer.toGeoJSON();
        const turfPolygon = turf.polygon(geoJson.geometry.coordinates);
        
        this.currentMeasurement = {
          area: turf.area(turfPolygon),
          perimeter: turf.length(turfPolygon, { units: 'meters' }),
          date: new Date()
        };

        this.currentPolygonPoints = geoJson.geometry.coordinates[0].map(coord => ({
          lat: coord[1],
          lng: coord[0]
        }));
      } catch (error) {
        console.error('Erro ao atualizar medição:', error);
      }
    },

    formatArea(area) {
      const unit = this.measurementUnits.area.find(u => u.value === this.selectedUnit);
      const converted = area * unit.conversion;
      return `${this.formatNumber(converted)} ${unit.label}`;
    },

    formatPerimeter(perimeter) {
      const unit = this.measurementUnits.perimeter.find(u => u.value === this.selectedUnit);
      const converted = perimeter * unit.conversion;
      return `${this.formatNumber(converted)} ${unit.label}`;
    },

    formatNumber(num, decimals = 2) {
      return new Intl.NumberFormat('pt-BR', {
        minimumFractionDigits: decimals,
        maximumFractionDigits: decimals
      }).format(num);
    },

    formatCoordinate(point) {
      return `${this.formatNumber(point.lat, 6)}°, ${this.formatNumber(point.lng, 6)}°`;
    },

    formatDate(dateString) {
      if (!dateString) return '';
      const date = new Date(dateString);
      return date.toLocaleDateString('pt-BR', {
        day: '2-digit',
        month: '2-digit',
        year: 'numeric',
        hour: '2-digit',
        minute: '2-digit'
      });
    },

    clearAll() {
      if (this.drawnItems) {
        this.drawnItems.clearLayers();
        this.currentMeasurement = null;
        this.currentPolygonPoints = [];
      }
    },

    saveMeasurement() {
      if (!this.currentMeasurement) return;
      
      // Validate measurement values before saving
      if (isNaN(this.currentMeasurement.area) || isNaN(this.currentMeasurement.perimeter)) {
        alert('Medição inválida. Por favor, tente novamente.');
        return;
      }
      
      const measurement = {
        ...this.currentMeasurement,
        coordinates: this.currentPolygonPoints.map(p => [p.lat, p.lng]),
        date: new Date().toISOString()
      };
      
      this.measurementHistory.unshift(measurement);
      localStorage.setItem('measurementHistory', JSON.stringify(this.measurementHistory));
      alert('Medição salva com sucesso!');
    },

    loadSavedMeasurements() {
      try {
        const saved = localStorage.getItem('measurementHistory');
        if (saved) {
          const measurements = JSON.parse(saved);
          // Filter out invalid measurements
          this.measurementHistory = measurements.filter(m => 
            !isNaN(m.area) && 
            !isNaN(m.perimeter) && 
            m.area > 0 && 
            m.perimeter > 0
          );
          // Update localStorage with clean data
          localStorage.setItem('measurementHistory', JSON.stringify(this.measurementHistory));
        }
      } catch (error) {
        console.error('Erro ao carregar medições:', error);
      }
    },

    loadMeasurement(item) {
      if (!item || !item.coordinates) return;
      
      this.clearAll();
      
      const polygon = L.polygon(item.coordinates, {
        color: '#2196F3',
        fillOpacity: 0.15,
        weight: 2
      });
      
      this.drawnItems.addLayer(polygon);
      this.currentMeasurement = {
        area: item.area,
        perimeter: item.perimeter,
        date: item.date
      };
      
      this.currentPolygonPoints = item.coordinates.map(coord => ({
        lat: coord[0],
        lng: coord[1]
      }));
      
      this.map.fitBounds(polygon.getBounds());
      this.showHistory = false;
    },

    toggleHistory() {
      this.showHistory = !this.showHistory;
    },

    setupKeyboardShortcuts() {
      document.addEventListener('keydown', (e) => {
        if (e.altKey) {
          switch (e.key.toLowerCase()) {
            case 'a': this.getArea(); break;
            case 'p': this.getPerimeter(); break;
            case 'c': this.clearAll(); break;
            case 'h': this.toggleHistory(); break;
            case 's': this.saveMeasurement(); break;
          }
        }
      });
    },

    changeLayer(layerId) {
      this.currentLayer = layerId;
    },

    updateMeasurements() {
      // Implementation needed
    },

    handleSearchInput() {
      if (this.searchTimeout) {
        clearTimeout(this.searchTimeout);
      }
      
      this.searchTimeout = setTimeout(() => {
        if (this.searchQuery.trim().length >= this.searchMinLength) {
          this.searchLocation();
        }
      }, this.searchDebounceTime);
    },

    async searchLocation() {
      if (!this.searchQuery.trim() || !this.map) return;
      
      this.isSearching = true;
      try {
        const query = encodeURIComponent(this.searchQuery.trim());
        const response = await fetch(`https://nominatim.openstreetmap.org/search?format=json&q=${query}&limit=1`);
        const data = await response.json();
        
        if (data && data.length > 0) {
          const location = data[0];
          this.map.setView([location.lat, location.lon], 18, {
            animate: true,
            duration: 1
          });
        } else {
          alert('Endereço não encontrado. Por favor, tente novamente com um endereço mais específico.');
        }
      } catch (error) {
        console.error('Erro na busca:', error);
        alert('Erro ao buscar endereço. Por favor, tente novamente.');
      } finally {
        this.isSearching = false;
      }
    },

    togglePrecisionMode() {
      this.isPrecisionMode = !this.isPrecisionMode;
      if (this.isPrecisionMode) {
        this.map.setZoom(MAP_CONFIG.ZOOM.PRECISION);
        this.gridLayer.setOpacity(0.6);
      } else {
        this.map.setZoom(MAP_CONFIG.ZOOM.DEFAULT);
        this.gridLayer.setOpacity(0.4);
      }
    },

    copyToClipboard(text) {
      const input = document.createElement('input');
      input.value = text;
      document.body.appendChild(input);
      input.select();
      document.execCommand('copy');
      document.body.removeChild(input);
      alert('Texto copiado para a área de transferência!');
    },

    async updateMapLayer(newLayer) {
      if (!this.map || !newLayer) return;
      
      try {
        await this.addTileLayer();
      } catch (error) {
        console.error('Erro ao mudar camada do mapa:', error);
        this.handleError(error, 'Erro ao mudar a visualização do mapa');
      }
    }
  }
};
</script>

<style scoped>
.property-map {
  width: 100%;
  max-width: 1400px;
  margin: 0 auto;
  padding: 20px;
  background: #f8f9fa;
  min-height: 100vh;
}

.brand {
  text-align: center;
  margin-bottom: 20px;
}

.brand h2 {
  margin: 0;
  color: #1a237e;
  font-size: 2.5em;
  font-weight: 700;
  letter-spacing: -0.5px;
}

.subtitle {
  color: #455a64;
  font-size: 1.1em;
  font-weight: 500;
}

.logo {
  display: flex;
  align-items: center;
  gap: 15px;
  justify-content: center;
}

.logo i {
  font-size: 2.5em;
  color: #1a237e;
  animation: rotate 20s linear infinite;
}

.instructions-panel {
  background: white;
  border-radius: 10px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.05);
  padding: 20px;
  margin-bottom: 30px;
}

.instructions-header {
  display: flex;
  align-items: center;
  gap: 10px;
  color: #1a237e;
  font-size: 1.2em;
  font-weight: 600;
  margin-bottom: 15px;
}

.instructions-content {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 20px;
}

.instruction-step {
  display: flex;
  align-items: flex-start;
  gap: 15px;
  padding: 10px;
  background: #f8f9fa;
  border-radius: 8px;
  transition: transform 0.2s;
}

.instruction-step:hover {
  transform: translateY(-2px);
}

.step-number {
  background: #1a237e;
  color: white;
  width: 24px;
  height: 24px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  flex-shrink: 0;
}

.map-wrapper {
  background: white;
  border-radius: 12px;
  box-shadow: 0 8px 16px rgba(0, 0, 0, 0.1);
  padding: 20px;
  margin-bottom: 30px;
}

.map-container {
  height: 600px;
  width: 100%;
  border-radius: 8px;
  overflow: hidden;
  position: relative;
}

.loading-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(255, 255, 255, 0.95);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.spinner {
  width: 50px;
  height: 50px;
  border: 4px solid #e3f2fd;
  border-top: 4px solid #1a237e;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

.info-panel {
  position: absolute;
  top: 80px;
  left: 20px;
  width: 320px;
  background: white;
  border-radius: 12px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  z-index: 1000;
  overflow: hidden;
}

.info-section {
  padding: 16px;
  border-bottom: 1px solid #e0e0e0;
}

.info-section:last-child {
  border-bottom: none;
}

.section-header {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 12px;
}

.section-header i {
  color: #1a237e;
  font-size: 1.2em;
}

.section-header h3 {
  margin: 0;
  font-size: 1.1em;
  color: #1a237e;
}

.measurement-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
  margin-bottom: 12px;
}

.measurement-item {
  background: #f8f9fa;
  padding: 12px;
  border-radius: 8px;
}

.measurement-label {
  display: flex;
  align-items: center;
  gap: 6px;
  color: #666;
  font-size: 0.9em;
  margin-bottom: 4px;
}

.measurement-value {
  font-size: 1.1em;
  font-weight: 600;
  color: #1a237e;
  display: flex;
  align-items: center;
  gap: 8px;
}

.selectable-value {
  flex: 1;
  user-select: text;
  cursor: text;
  padding: 8px;
  background: rgba(26, 35, 126, 0.05);
  border-radius: 4px;
  transition: all 0.2s;
  border: none;
  font-size: 1em;
  font-weight: 600;
  color: #1a237e;
  width: 100%;
  text-align: right;
}

.selectable-value:hover {
  background: rgba(26, 35, 126, 0.1);
}

.selectable-value:focus {
  outline: 2px solid rgba(26, 35, 126, 0.3);
  background: white;
}

.copy-button {
  background: none;
  border: none;
  color: #1a237e;
  padding: 4px 8px;
  cursor: pointer;
  border-radius: 4px;
  transition: all 0.2s;
}

.copy-button:hover {
  background: rgba(26, 35, 126, 0.1);
}

.copy-button:active {
  transform: scale(0.95);
}

.coordinates-list {
  max-height: 200px;
  overflow-y: auto;
}

.coordinate-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 8px;
  border-bottom: 1px solid #f0f0f0;
}

.coordinate-item:last-child {
  border-bottom: none;
}

.point-number {
  background: #1a237e;
  color: white;
  width: 24px;
  height: 24px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.9em;
  font-weight: 600;
}

.coordinate {
  font-family: monospace;
  color: #333;
}

.toolbar {
  display: flex;
  gap: 20px;
  margin: 20px auto;
  align-items: center;
  max-width: 1200px;
  padding: 20px;
  background: #ffffff;
  border-radius: 12px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
}

.search-bar {
  flex: 1;
  position: relative;
  max-width: 600px;
  margin: 0 auto;
}

.search-bar input {
  width: 100%;
  padding: 15px 45px;
  border: 2px solid #e0e0e0;
  border-radius: 25px;
  font-size: 1.1em;
  transition: all 0.3s;
  background: white;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.05);
}

.search-bar input:focus {
  border-color: #1a237e;
  outline: none;
  box-shadow: 0 4px 12px rgba(26, 35, 126, 0.15);
  transform: translateY(-1px);
}

.search-bar input::placeholder {
  color: #9e9e9e;
}

.search-bar i {
  position: absolute;
  left: 16px;
  top: 50%;
  transform: translateY(-50%);
  color: #1a237e;
  font-size: 1.2em;
  pointer-events: none;
}

.tools {
  display: flex;
  gap: 10px;
  align-items: center;
}

.tool-button {
  padding: 10px;
  border-radius: 8px;
  background: white;
  border: 2px solid #e0e0e0;
  color: #1a237e;
  transition: all 0.3s;
}

.tool-button:hover {
  background: #f5f5f5;
  border-color: #1a237e;
}

.map-overlay-controls {
  position: absolute;
  top: 20px;
  right: 20px;
  display: flex;
  flex-direction: row;
  gap: 12px;
  z-index: 999;
}

.layer-controls {
  background: white;
  border-radius: 8px;
  padding: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  display: flex;
  flex-direction: row;
  gap: 4px;
}

.layer-controls button {
  padding: 8px;
  min-width: 36px;
  height: 36px;
  border: none;
  background: none;
  color: #666;
  cursor: pointer;
  transition: all 0.2s;
  border-radius: 4px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.layer-controls button:hover {
  background: #f5f5f5;
  color: #1a237e;
}

.layer-controls button.active {
  background: #1a237e;
  color: white;
}

.measurement-history {
  position: absolute;
  top: 80px;
  right: 20px;
  background: white;
  border-radius: 12px;
  width: 300px;
  max-height: calc(100vh - 120px);
  overflow-y: auto;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  z-index: 1000;
}

.history-item {
  padding: 15px;
  border-bottom: 1px solid #eee;
  cursor: pointer;
  transition: all 0.2s;
}

.history-item:hover {
  background: #f5f5f5;
}

.history-date {
  color: #666;
  font-size: 0.9em;
}

.history-values {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-top: 8px;
}

.history-value {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 0.95em;
}

.history-value i {
  color: #1a237e;
  width: 16px;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

@keyframes rotate {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

@media (max-width: 768px) {
  .instructions-content {
    grid-template-columns: 1fr;
  }

  .toolbar {
    flex-direction: column;
    padding: 15px;
  }

  .search-bar {
    width: 100%;
  }

  .tools {
    justify-content: center;
    flex-wrap: wrap;
  }

  .info-panel {
    top: auto;
    bottom: 20px;
    left: 50%;
    transform: translateX(-50%);
    width: 90%;
    max-width: 320px;
  }

  .measurement-history {
    top: auto;
    bottom: 20px;
    right: 50%;
    transform: translateX(50%);
    width: 90%;
    max-width: 300px;
  }
}

.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}

.search-loading {
  position: absolute;
  right: 16px;
  top: 50%;
  transform: translateY(-50%);
  color: #1a237e;
}

.layer-selector {
  display: none;
}

.map-layer-selector {
  position: absolute;
  bottom: 25px;
  right: 25px;
  z-index: 1000;
  background: white;
  border-radius: 8px;
  padding: 6px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
}

.layer-controls {
  display: flex;
  gap: 4px;
}

.layer-button {
  width: 36px;
  height: 36px;
  border: none;
  background: white;
  color: #666;
  border-radius: 4px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.2s ease;
}

.layer-button:hover {
  background: #f5f5f5;
  color: #1a237e;
  transform: translateY(-1px);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.layer-button.active {
  background: #1a237e;
  color: white;
}

.layer-button:active {
  transform: translateY(1px);
}

.signature {
  text-align: center;
  padding: 20px;
  margin-top: 20px;
  color: #455a64;
  font-size: 0.9em;
  border-top: 1px solid #e0e0e0;
}

.signature p {
  margin: 5px 0;
}

.signature p:first-child {
  font-weight: 500;
}
</style>