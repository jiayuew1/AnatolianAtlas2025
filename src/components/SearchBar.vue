<!-- SearchBar.vue component -->
<template>
  <div class="search-container">
    <div class="search-input-wrapper">
      <input 
        type="text" 
        v-model="searchQuery" 
        @input="handleSearch"
        placeholder="Search by site name or AA_ID..." 
        class="search-input"
      />
      <button 
        v-if="searchQuery" 
        @click="clearSearch" 
        class="clear-search-button"
      >
        ×
      </button>
    </div>
    <div v-if="searchResults.length > 0" class="search-results">
      <div 
        v-for="result in searchResults" 
        :key="result.id" 
        @click="selectSite(result)"
        class="search-result-item"
      >
        <div class="result-name">{{ result.name }}</div>
        <div class="result-survey">{{ result.survey }}</div>
        <div class="result-id" v-if="result.id">ID: {{ result.id }}</div>
      </div>
    </div>
    
    <!-- Display when no results found -->
    <div v-if="searchQuery && searchResults.length === 0" class="no-results">
      No sites found {{ isPeriodFilterActive ? 'within selected periods' : '' }}
    </div>
    
    <!-- Display current selections -->
    <div v-if="selectedSites.length > 0" class="current-selections">
      <div class="selections-header">
        <span>Current Selections ({{ selectedSites.length }})</span>
        <button @click="clearAllSelections" class="clear-all-button">Clear All</button>
      </div>
      <div class="selected-items">
        <div 
          v-for="site in selectedSites" 
          :key="site.id" 
          class="selected-item"
        >
        <span class="selected-name">
        {{
            site.data.AA_ID
            ? site.data.AA_ID + (getAACount(site.data.AA_ID) > 1 ? '(' + getAACount(site.data.AA_ID) + ')' : '')
            : site.id
        }}
        </span>
          <button @click="removeSite(site)" class="remove-button">×</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'SearchBar',
  props: {
    points: {
      type: Array,
      required: true
    },
    // New prop to receive period-filtered points
    filteredByPeriod: {
      type: Array,
      default: () => []
    },
    // New prop to know if period filtering is active
    isPeriodFilterActive: {
      type: Boolean,
      default: false
    }
  },
  data() {
    return {
      searchQuery: '',
      searchResults: [],
      selectedSites: [], // Track selected sites locally
      turkishReplacements: {
        'i': 'ıiİI',
        'o': 'oöOÖ',
        'u': 'uüUÜ',
        's': 'sşSŞ',
        'g': 'gğGĞ',
        'c': 'cçCÇ',
        'h': 'hHĥĤ',
        'a': 'aâAÂ'
      }
    };
  },
  methods: {
    getAACount(aaid) {
        return this.points.filter(point => point.AA_ID === aaid).length;
    },
    handleSearch() {
      if (!this.searchQuery.trim()) {
        this.searchResults = [];
        return;
      }
      
      const normalizedQuery = this.normalizeText(this.searchQuery.toLowerCase().trim());
      
      // Use filteredByPeriod points if period filters are active, otherwise use all points
      const pointsToSearch = this.isPeriodFilterActive && this.filteredByPeriod.length > 0 
        ? this.filteredByPeriod 
        : this.points;
      
      // Search with fuzzy Turkish character matching
      this.searchResults = pointsToSearch
        .filter(point => {
            const name = point.Name || '';
            const altNames = point.AlternativeNames || '';
            const id = (point.AA_ID || point.OBJECTID || '').toString();

            const normalizedName = this.normalizeText(name.toLowerCase());
            const normalizedAltNames = this.normalizeText(altNames.toLowerCase());
            const normalizedId = this.normalizeText(id.toLowerCase());

            return normalizedName.includes(normalizedQuery) || 
                normalizedAltNames.includes(normalizedQuery) ||
                normalizedId.includes(normalizedQuery);
        })
        .map(point => ({
          id: point.AA_ID || point.OBJECTID,
          name: point.Name || 'Unnamed Site',
          survey: point.Survey || 'Unknown Survey',
          data: point
        }))
        .slice(0, 10); // Limit to 10 results
        
      // Debug logging
      console.log(`Found ${this.searchResults.length} results for "${normalizedQuery}" ${this.isPeriodFilterActive ? '(period filtered)' : ''}`);
    },
    
    normalizeText(text) {
      // Create a normalized version of the text with Turkish character variants replaced
      let normalized = text;
      
      // Replace Turkish characters with their Latin equivalents for searching
      for (const [latin, variants] of Object.entries(this.turkishReplacements)) {
        const regex = new RegExp(`[${variants}]`, 'g');
        normalized = normalized.replace(regex, latin);
      }
      
      return normalized;
    },
    
    selectSite(result) {
      // Check if already selected
      const isAlreadySelected = this.selectedSites.some(site => site.id === result.id);
      
      if (!isAlreadySelected) {
        // Add to local selected sites
        this.selectedSites.push(result);
        
        // Always emit the site selected event
        // We ignore whether it was from period-filtered results
        this.$emit('site-selected', result.data);
        
        // Log to help debug
        console.log('Selected site:', result.id);
      }
      
      // Clear search results but keep query
      this.searchResults = [];
    },
    
    removeSite(site) {
      // Remove from local selection
      this.selectedSites = this.selectedSites.filter(s => s.id !== site.id);
      
      // Emit to parent
      this.$emit('site-removed', site.id);
    },
    
    clearSearch() {
      this.searchQuery = '';
      this.searchResults = [];
    },
    
    clearAllSelections() {
      this.selectedSites = [];
      this.$emit('search-cleared');
    }
  },
  // Add a watch for filteredByPeriod changes to refresh search if needed
  watch: {
    filteredByPeriod() {
      // Re-run search if there's an active query and period filters changed
      if (this.searchQuery.trim()) {
        this.handleSearch();
      }
    },
    // Also watch for changes in isPeriodFilterActive
    isPeriodFilterActive() {
      // Clear search results when period filter status changes
      this.searchResults = [];
      if (this.searchQuery.trim()) {
        this.handleSearch();
      }
    }
  }
};
</script>

<style scoped>
.search-container {
  margin: 16px 0;
  position: relative;
}

.search-input-wrapper {
  position: relative;
  display: flex;
  align-items: center;
}

.search-input {
  width: 100%;
  padding: 8px 30px 8px 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
}

.clear-search-button {
  position: absolute;
  right: 8px;
  background: none;
  border: none;
  font-size: 18px;
  cursor: pointer;
  color: #999;
  padding: 0;
  width: 20px;
  height: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.clear-search-button:hover {
  color: #333;
}

.search-results {
  position: absolute;
  width: 100%;
  max-height: 300px;
  overflow-y: auto;
  background: white;
  border: 1px solid #ddd;
  border-top: none;
  border-radius: 0 0 4px 4px;
  z-index: 1000;
  box-shadow: 0 4px 8px rgba(0,0,0,0.1);
}

.search-result-item {
  padding: 10px;
  cursor: pointer;
  border-bottom: 1px solid #eee;
}

.search-result-item:last-child {
  border-bottom: none;
}

.search-result-item:hover {
  background-color: #f5f5f5;
}

.result-name {
  font-weight: bold;
  margin-bottom: 3px;
}

.result-survey {
  font-size: 12px;
  color: #666;
}

.result-id {
  font-size: 11px;
  color: #888;
  margin-top: 2px;
}

.no-results {
  padding: 8px;
  color: #666;
  font-size: 14px;
  text-align: center;
  background: #f8f8f8;
  margin-top: 4px;
  border-radius: 4px;
}

/* Selected sites styling */
.current-selections {
  margin-top: 16px;
  border: 1px solid #eee;
  border-radius: 4px;
  overflow: hidden;
}

.selections-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 10px;
  background: #f5f5f5;
  border-bottom: 1px solid #eee;
  font-size: 13px;
  font-weight: bold;
}

.clear-all-button {
  background: none;
  border: none;
  color: #888;
  font-size: 12px;
  cursor: pointer;
  padding: 2px 5px;
}

.clear-all-button:hover {
  color: #333;
  text-decoration: underline;
}

.selected-items {
  max-height: 150px;
  overflow-y: auto;
}

.selected-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 6px 10px;
  border-bottom: 1px solid #eee;
  font-size: 13px;
}

.selected-item:last-child {
  border-bottom: none;
}

.selected-name {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.remove-button {
  background: none;
  border: none;
  color: #999;
  cursor: pointer;
  padding: 0;
  width: 18px;
  height: 18px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 14px;
  border-radius: 50%;
}

.remove-button:hover {
  background: #f0f0f0;
  color: #333;
}
</style>