<template>
  <div class="period-tree" :style="{ marginLeft: level ? '24px' : '0' }">
    <div v-for="(value, period) in periods" :key="period" class="period-group">
      <div class="period-item">
        <input 
          type="checkbox" 
          :id="period"
          :checked="selectedPeriods[period]"
          @change="handleCheckboxChange(period, $event.target.checked)"
        >
        <label :for="period">{{ formatDisplayName(period) }}</label>
        <span class="spacer"></span>
        <span 
          v-if="Object.keys(value).length > 0"
          class="expand-btn"
          @click="$emit('togglePeriod', period)"
        >
          {{ expandedPeriods[period] ? '-' : '+' }}
        </span>
      </div>
      <div v-if="expandedPeriods[period] && Object.keys(value).length > 0">
        <period-tree
          :periods="value"
          :selected-periods="selectedPeriods"
          :expanded-periods="expandedPeriods"
          :level="level + 1"
          @update-period="handleSubPeriodChange"
          @toggle-period="$emit('togglePeriod', $event)"
        />
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'PeriodTree',
  props: {
    periods: {
      type: Object,
      required: true
    },
    selectedPeriods: {
      type: Object,
      required: true
    },
    expandedPeriods: {
      type: Object,
      required: true
    },
    level: {
      type: Number,
      default: 0
    }
  },
  emits: ['updatePeriod', 'togglePeriod'],
   methods: {
    formatDisplayName(name) {
      if (!name) return '';
      name = name.replace(/PrePottery/g, 'Pre-Pottery');
      return name
        .replace(/([A-Z][a-z]+)([IVX]+)/g, '$1 $2')  // Handle Roman numerals
        .replace(/([a-z])([A-Z])/g, '$1 $2')         // Add space between lower and uppercase
        .replace(/(?<!-)([A-Z][a-z])/g, ' $1')       // Add space before capital letters, but not after hyphen
        .trim()
        .replace(/\s+/g, ' ');                       // Clean up multiple spaces
    },
    handleCheckboxChange(period, checked) {
      // Keep the original period ID for internal logic
      this.$emit('updatePeriod', period, checked)
    },
    handleSubPeriodChange(period, checked) {
      // Keep the original period ID for internal logic
      this.$emit('updatePeriod', period, checked)
    }
  }
}

</script>
<style scoped>
.period-tree {
  font-size: 14px;
}

.period-item {
  display: flex;
  align-items: center;
  margin: 4px 0;
  gap: 8px;  /* This ensures minimum spacing between elements */
  padding-right: 8px;  /* Add some padding on the right */
  width: 100%;  /* Ensure the item takes full width */
}

.expand-btn {
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

.expand-btn:hover {
  background-color: #f5f5f5;  /* Light background on hover */
  color: #333;
}

.spacer {
  flex: 1;
  min-width: 20px;  /* Minimum space between text and button */
}

label {
  cursor: pointer;
  user-select: none;
}

input[type="checkbox"] {
  cursor: pointer;
}
</style>