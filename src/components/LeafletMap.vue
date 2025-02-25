<template>
  <div class="container">
    <!-- Map Section -->
    <div class="map-section">
      <cluster-map
        :points="filteredPoints"
        :get-color="getSurveyColor"
      />
      <!-- Legend inside map -->
      <div class="legend-control">
      <h3 @click="toggleLegend" class="legend-header">
        Surveys
        <span class="spacer"></span>
        <span class="toggle-icon">{{ isLegendCollapsed ? '+' : '-' }}</span>
      </h3>
        <div class="legend-items" v-show="!isLegendCollapsed">
          <div v-for="(color, survey) in surveyColors" :key="survey" class="legend-item">
            <span class="color-dot" :style="{ backgroundColor: color }"></span>
            <span class="survey-name">{{ formatDisplayName(survey) }}</span>
          </div>
        </div>
      </div>
    </div>

    <!-- Filter Panel -->
    <div class="filter-panel">
      <h3>Period Filters ({{ filteredPoints.length }} points shown)</h3>
      <div class="filter-tree">
        <PeriodTree
          :periods="periods"
          :selectedPeriods="selectedPeriods"
          :expandedPeriods="expandedPeriods"
          @togglePeriod="togglePeriod"
          @updatePeriod="handlePeriodChange"
        />
      </div>
      <div class="clear-button-container">
        <button class="clear-button" @click="clearSelections">
          Clear Selections
        </button>
        <button class="download-button" @click="downloadSelections">
          Download Current Selections
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import Papa from 'papaparse'
import PeriodTree from './PeriodTree.vue'
import ClusterMap from './MarkerCluster.vue'
import * as XLSX from 'xlsx'

export default {
  name: 'LeafletMap',
  components: {
    PeriodTree,
    ClusterMap
  },
  data() {
    return {
      isLegendCollapsed: false,
      points: [],
      dataStatus: 'Initializing...',
      errorMessage: '',
      selectedPeriods: {},
      expandedPeriods: {},
      surveyColors: {},
      filterCache: new Map(),
      periods: {
        "Paleolithic": {
          "LowerPaleolithic": {},
          "MiddlePaleolithic": {}
        },
        "PrePotteryNeolithic": {
          "PrePotteryNeolithicA": {},
          "PrePotteryNeolithicB": {}
        },
        "Neolithic": {},
        "Chalcolithic": {
          "EarlyChalcolithic": {
            "Halaf": {}
          },
          "MiddleChalcolithic": {},
          "LateChalcolithic": {
            "Uruk": {},
            "LateUruk": {}
          }
        },
        "BronzeAge": {
          "EarlyBronzeAge": {
            "EarlyBronzeAgeI": {},
            "EarlyBronzeAgeII": {},
            "EarlyBronzeAgeIII": {},
            "EarlyBronzeAgeIV": {}
          },
          "MiddleBronzeAge": {
            "MiddleBronzeAgeI": {},
            "MiddleBronzeAgeII": {}
          },
          "LateBronzeAge": {
            "OldHittite": {},
            "EarlyNewEmpire": {},
            "LateNewEmpire": {}
          }
        },
        "IronAge": {
          "EarlyIronAge": {},
          "MiddleIronAge": {},
          "Archaic": {},
          "LateIronAge": {}
        },
        "Classical": {
          "Achaemenid": {},
          "Hellenistic": {
            "EarlyHellenistic": {},
            "LateHellenistic": {}
          },
          "Roman": {
            "EarlyRoman": {},
            "LateRoman": {}
          }
        },
        "Medieval": {
          "LateAntiquity": {},
          "Byzantine": {
            "EarlyByzantine": {},
            "MiddleByzantine": {}
          },
          "Turkic": {},
          "Islamic": {
            "EarlyIslamic": {},
            "MiddleIslamic": {},
            "Abbasid": {}
          },
          "Armenian": {},
          "Mamluk": {}
        },
        "Ottoman": {}
      },
      colors: [
        '#2E86AB',    // Steel blue
        '#D64933',    // Coral red
        '#4B7F52',    // Forest green
        '#9B4F96',    // Plum purple
        '#E5B731',    // Golden yellow
        '#45B7D1',    // Turquoise
        '#BA5C3D',    // Rust
        '#5C6F7B',    // Slate
        '#8E9B4B',    // Olive
        '#7C53A2',    // Royal purple
        '#D3843D',    // Bronze
        '#498591',    // Teal
        '#B15DAA',    // Magenta
        '#687A3D',    // Moss green
        '#8B6C8F'     // Mauve
      ]
    }
  },
  computed: {
    filteredPoints() {
      const selectedPeriodNames = Object.entries(this.selectedPeriods)
        .filter(([, isSelected]) => isSelected)
        .map(([period]) => period);

      if (selectedPeriodNames.length === 0) {
        return this.points;
      }

      const cacheKey = selectedPeriodNames.sort().join('|');
      
      if (this.filterCache.has(cacheKey)) {
        return this.filterCache.get(cacheKey);
      }

      const filtered = this.points.filter(point => {
        if (!point.News) return false;
        const pointPeriods = point.News.split(',').map(p => p.trim());
        return selectedPeriodNames.some(period => pointPeriods.includes(period));
      });

      this.filterCache.set(cacheKey, filtered);
      return filtered;
    }
  },
  methods: {
    toggleLegend() {
      this.isLegendCollapsed = !this.isLegendCollapsed; // Toggle the collapsed state
    },
    formatDisplayName(name) {
      if (!name) return '';
      return name 
        .replace(/([A-Z][a-z]+)([IVX]+)/g, '$1 $2')
        .replace(/([a-z])([A-Z])/g, '$1 $2')
        .replace(/([A-Z][a-z])/g, ' $1')
        .trim()
        .replace(/\s+/g, ' ')
        .replace(/_/g, ' ') ;
    },
    clearSelections() {
      this.filterCache.clear();
      Object.keys(this.selectedPeriods).forEach(period => {
        this.selectedPeriods[period] = false;
      });
    },
    getSurveyColor(survey) {
      return this.surveyColors[survey] || '#999999';
    },
    togglePeriod(period) {
      this.expandedPeriods[period] = !this.expandedPeriods[period];
    },
    handlePeriodChange(period, checked) {
      this.selectedPeriods[period] = checked;
      
      const updateSubPeriods = (periodsObj, parentChecked) => {
        for (const [key, value] of Object.entries(periodsObj)) {
          this.selectedPeriods[key] = parentChecked;
          if (Object.keys(value).length > 0) {
            updateSubPeriods(value, parentChecked);
          }
        }
      };
      
      if (this.periods[period]) {
        updateSubPeriods(this.periods[period], checked);
      }
    },
    
    downloadSelections() {
      try {
        // Create a new workbook
        const wb = XLSX.utils.book_new();
        
        // Select and rename columns for the filtered points
        const selectedColumns = this.filteredPoints.map(point => ({
          OBJECTID: point.OBJECTID,
          'Survey Name': point['Survey'],
          'Site Name': point['Name'],
          'Alternative Names': point.AlternativeNames,
          'Occupational Periods': point.News,
          x: point.x,
          y: point.y
        }));
        
        // Convert to sheet with selected columns
        const ws = XLSX.utils.json_to_sheet(selectedColumns);
        
        // Append sheet to workbook
        XLSX.utils.book_append_sheet(wb, ws, "Selected Surveys");
        
        // Create file name with timestamp
        const timestamp = new Date().toISOString().replace(/[:.]/g, '-').slice(0, 19);
        const fileName = `selected_surveys_${timestamp}.xlsx`;
      
        // Save the document 
        XLSX.writeFile(wb, fileName);
      } catch (error) {
        console.error('Error downloading Excel file:', error);
      }
    },

    created() {
    // Initialize selectedPeriods and expandedPeriods
    const initializePeriods = (periodsObj) => {
      for (const [key, value] of Object.entries(periodsObj)) {
        this.selectedPeriods[key] = false;
        this.expandedPeriods[key] = false;
        if (Object.keys(value).length > 0) {
          initializePeriods(value);
        }
      }
    };
    
    initializePeriods(this.periods);
  },

  },
  async mounted() {
    try {
      this.dataStatus = 'Loading CSV...';
      
      const response = await fetch('/all_surveys_Oct2024_points_0.csv');
      
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      
      // Read ArrayBuffer 
      const buffer = await response.arrayBuffer();
      const encodings = ['utf-8', 'windows-1254', 'iso-8859-9'];
      let csvText;
      
      for (const encoding of encodings) {
        try {
          const decoder = new TextDecoder(encoding);
          const text = decoder.decode(buffer);
          // Check for special letters
          if (text.includes('ç') || text.includes('ı') || text.includes('ğ')) {
            csvText = text;
            console.log(`Successfully decoded using ${encoding}`);
            break;
          }
        } catch (e) {
          console.log(`Failed to decode with ${encoding}:`, e);
        }
      }
      
      if (!csvText) {
        csvText = new TextDecoder('utf-8').decode(buffer);
        console.log('Falling back to UTF-8');
      }
      
      this.dataStatus = 'Parsing CSV...';
      
      Papa.parse(csvText, {
        header: true,
        skipEmptyLines: true,
        complete: (results) => {
          // 检查解析结果
          console.log('Sample of parsed data:', results.data.slice(0, 2));
          
          this.points = results.data.filter(row => {
            const hasCoordinates = row.x && row.y;
            const validCoordinates = !isNaN(parseFloat(row.x)) && !isNaN(parseFloat(row.y));
            return hasCoordinates && validCoordinates;
          });
          
          const uniqueSurveys = [...new Set(this.points.map(p => p.Survey))];
          uniqueSurveys.forEach((survey, index) => {
            this.surveyColors[survey] = this.colors[index % this.colors.length];
          });

          this.dataStatus = `Parsed ${this.points.length} points`;
          
          if (this.points.length > 0) {
            //check for Turkish letters
            const turkishChars = this.points.some(p => 
              Object.values(p).some(v => 
                String(v).match(/[çğıöşüÇĞİÖŞÜ]/)
              )
            );
            console.log('Contains Turkish characters:', turkishChars);
            console.log('First point data:', this.points[0]);
          }
        },
        error: (error) => {
          console.error('CSV parsing error:', error);
          this.errorMessage = `CSV parsing error: ${error.message}`;
          this.dataStatus = 'Error parsing CSV';
        }
      });
    } catch (error) {
      console.error('File loading error:', error);
      this.errorMessage = `File loading error: ${error.message}`;
      this.dataStatus = 'Error loading file';
    }
  }
};
</script>


<style scoped>

.container {
  display:flex;
  height: 100vh;  
  width: 100%;
  flex-direction: row;
}

.map-section {
  flex: 1;
  position: relative;
  height: 100vh;
}

.filter-panel {
  top: 0;
  left: 0;
  z-index: 1000;
  width: 250px;
  height: 100vh;
  padding: 20px;
  overflow-y: auto;
  border-right: 1px solid #eee;
  background: white;
  flex-shrink: 0;
  order: -1;
}

.filter-panel h3 {
  margin: 0 0 16px 0;
  font-size: 16px;
}
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

/* Right survey legend */
.legend-control {
  position: absolute;
  top: 20px;
  right: 20px;
  z-index: 1000;
  background: white;
  padding: 20px;
  border-radius: 5px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
  max-width: 200px;
  max-height: calc(100vh - 40px);
  overflow-y: auto;
}

.legend-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 16px;
  margin: 0; 
  padding: 0; 
  }


.legend-items {
  max-height: 300px;
  overflow-y: auto;
}

.legend-item {
  display: flex;
  align-items: center;
  margin: 2px 0;
  font-size: 12px;
  white-space: nowrap;
}

.color-dot {
  width: 12px;
  height: 12px;
  border-radius: 50%;
  margin-right: 6px;
  flex-shrink: 0;
}

.survey-name {
  overflow: hidden;
  text-overflow: ellipsis;
}

.expand-btn {
  cursor: pointer;
  color: #1a2eb1;
  font-size: 16px;
  width: 20px;
  height: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  user-select: none;
}

.expand-btn:hover {
  color: #ffffff;
}

.toggle-icon{
  cursor: pointer;
  color: #1a2eb1;
  font-size: 16px;
  width: 24px;  /* Fixed width for the button */
  height: 24px;  /* Fixed height for the button */
  display: flex;
  align-items: center;
  justify-content: center;
  user-select: none;
  margin-left: auto;  /* Push the button to the right */
  border: 1px solid #ddd;  /* Add a subtle border */
  border-radius: 4px;  /* Rounded corners */
}

.toggle-icon:hover {
  color: #ffffff;
}

.spacer {
  flex: 1;
  min-width: 10px;  /* Minimum space between text and button */
}

.sub-periods {
  margin-left: 24px;
}

.truncate {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}


.clear-button-container {
  display: flex;
  flex-direction: column;  
  gap: 8px;  
  padding: 16px 0;
  margin-top: 8px;
  border-top: 1px solid #eee;
}

.clear-button, .download-button {
  width: 100%;  
  padding: 8px 16px;
  background-color: white;
  border: 1px solid #ddd;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
  color: #333;
  transition: all 0.2s ease;
}

.clear-button:hover, .download-button:hover {
  background-color: #f5f5f5;
  border-color: #ccc;
}

</style>