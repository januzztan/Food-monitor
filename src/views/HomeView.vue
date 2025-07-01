<script setup>
import { onMounted, onBeforeUnmount, ref, computed } from 'vue'
import Highcharts from 'highcharts'
import { db } from '@/firebase'
import { collection, query, onSnapshot, orderBy } from 'firebase/firestore'

// =============================================
// CONFIGURATION & SETUP
// =============================================

// Set Highcharts global options
Highcharts.setOptions({
  lang: {
    decimalPoint: '.',
    thousandsSep: ',',
  },
})

// =============================================
// REACTIVE STATE
// =============================================

// Current sensor readings
const currentTemperature = ref(0)
const currentHumidity = ref(0)
const currentPressure = ref(0)
const currentDate = ref('--/--/----')
const currentTimeOnly = ref('--:--')
const currentTime = ref('--:--') // For compatibility

// Chart data arrays
const timeCategories = ref([])
const temperatureData = ref([])
const humidityData = ref([])
const pressureData = ref([])

// Data filtering
const startDate = ref('')
const endDate = ref('')
const allSensorData = ref([])
const isFilterActive = ref(false)

// Chart management variables
let charts = []
let combinedChart = null
let eventListeners = []
let unsubscribeFirestore = null
let isResetting = false // Track reset operations to prevent infinite loops

// =============================================
// SENSOR DATA CONFIGURATION
// =============================================

const sensorData = {
  temperature: {
    data: temperatureData,
    color: 'rgb(248, 113, 113)', // Softer red
    unit: '°C',
  },
  humidity: {
    data: humidityData,
    color: 'rgb(96, 165, 250)', // Softer blue
    unit: '%',
  },
  pressure: {
    data: pressureData,
    color: 'rgb(74, 222, 128)', // Softer green
    unit: 'hPa',
  },
}

// =============================================
// COMPUTED PROPERTIES
// =============================================

// Filtered data based on date range
const filteredSensorData = computed(() => {
  if (!isFilterActive.value || !startDate.value || !endDate.value) {
    return allSensorData.value
  }

  // Convert string dates to Date objects for comparison
  const start = new Date(startDate.value + ' 00:00:00')
  const end = new Date(endDate.value + ' 23:59:59')

  return allSensorData.value.filter(reading => {
    if (!reading.timestamp) return false

    // Convert timestamp string to Date object (assuming format: DD/MM/YYYY HH:MM:SS)
    const [datePart, timePart] = reading.timestamp.split(' ')
    const [day, month, year] = datePart.split('/')
    const dateStr = `${year}-${month}-${day}T${timePart}`
    const readingDate = new Date(dateStr)

    return readingDate >= start && readingDate <= end
  })
})

// =============================================
// DATA PROCESSING FUNCTIONS
// =============================================

// Process and display data (used by both filter and initial load)
function processAndDisplayData(data) {
  if (!data.length) return

  // Extract and prepare data for charts
  timeCategories.value = data.map((reading) => {
    // Extract both date and time parts from timestamp (format: 22/05/2025 14:11:10)
    if (!reading.timestamp) return '';

    const [datePart, timePart] = reading.timestamp.split(' ');
    const shortTime = timePart.substring(0, 5); // Get HH:MM

    // Format date as DD/MM
    const shortDate = datePart.split('/').slice(0, 2).join('/');

    return `${shortDate} ${shortTime}`;
  })

  temperatureData.value = data.map((reading) => reading.temperature || 0)
  humidityData.value = data.map((reading) => reading.humidity || 0)
  pressureData.value = data.map((reading) => reading.pressure || 0)

  // Update current readings with latest data
  updateCurrentReadings(data)

  // Update charts if they exist
  updateCharts()
}

// Update current readings display with latest data
function updateCurrentReadings(data) {
  if (!data || !data.length) return

  // Get the last element (most recent) if ordered by datetime ascending
  const latest = data[data.length - 1]

  // Update current values
  currentTemperature.value = latest.temperature?.toFixed(1) || 0
  currentHumidity.value = latest.humidity?.toFixed(1) || 0
  currentPressure.value = latest.pressure?.toFixed(1) || 0

  // Split timestamp into date and time parts
  if (latest.timestamp) {
    const [datePart, timePart] = latest.timestamp.split(' ');
    currentDate.value = datePart || '--/--/----';
    currentTimeOnly.value = timePart || '--:--';
  } else {
    currentDate.value = '--/--/----';
    currentTimeOnly.value = '--:--';
  }

  // Keep the original timestamp for compatibility
  currentTime.value = latest.timestamp || '--:--'
}

// =============================================
// FILTER FUNCTIONS
// =============================================

// Apply date filter
function applyDateFilter() {
  if (!startDate.value || !endDate.value) {
    alert('Please select both start and end dates')
    return
  }

  isFilterActive.value = true
  processAndDisplayData(filteredSensorData.value)
}

// Reset filters
function resetFilters() {
  startDate.value = ''
  endDate.value = ''
  isFilterActive.value = false
  processAndDisplayData(allSensorData.value)
}

// =============================================
// EXPORT FUNCTIONS
// =============================================

// Handle CSV export
function exportToCSV() {
  // Prepare the data to be exported
  const dataToExport = isFilterActive.value ? filteredSensorData.value : allSensorData.value;

  if (!dataToExport.length) {
    alert('No data available to export');
    return;
  }

  // Create CSV header row
  const headers = ['Timestamp', 'Date', 'Time', 'Temperature (°C)', 'Humidity (%)', 'Pressure (hPa)'];

  // Create CSV rows
  const rows = dataToExport.map(reading => {
    const [datePart, timePart] = reading.timestamp ? reading.timestamp.split(' ') : ['--/--/----', '--:--:--'];
    return [
      reading.timestamp || '--',
      datePart,
      timePart,
      reading.temperature?.toFixed(1) || '0',
      reading.humidity?.toFixed(1) || '0',
      reading.pressure?.toFixed(1) || '0'
    ].join(',');
  });

  // Combine header and rows
  const csvContent = [headers.join(','), ...rows].join('\n');

  // Create a Blob containing the CSV data with UTF-8 BOM
  const BOM = '\uFEFF'; // UTF-8 BOM
  const blob = new Blob([BOM + csvContent], { type: 'text/csv;charset=utf-8;' });

  // Create a download link and trigger download
  const link = document.createElement('a');

  // Create filename using current date/time
  const now = new Date();
  const fileName = isFilterActive.value
    ? `food-monitor-data-${startDate.value}-to-${endDate.value}.csv`
    : `food-monitor-data-${now.toISOString().split('T')[0]}.csv`;

  // Create download link with generated file name
  if (navigator.msSaveBlob) { // IE and Edge
    navigator.msSaveBlob(blob, fileName);
  } else {
    // For other browsers
    const url = URL.createObjectURL(blob);
    link.setAttribute('href', url);
    link.setAttribute('download', fileName);
    link.style.visibility = 'hidden';
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
  }
}

// =============================================
// CHART CONFIGURATION FUNCTIONS
// =============================================

// Get configuration options for individual charts
function getChartOptions(id, title, data) {
  const { color, unit, data: values } = data

  return {
    chart: {
      renderTo: id,
      type: 'line',
      zoomType: 'x',
      events: {
        selection: syncExtremes,
      },
      resetZoomButton: {
        position: {
          align: 'right',
          x: -10,
          y: 10
        },
        theme: {
          fill: 'white',
          stroke: '#999',
          r: 5,
          style: {
            color: '#003399'
          }
        }
      }
    },
    title: { text: title },
    xAxis: {
      categories: timeCategories.value,
      crosshair: true,
      events: {
        setExtremes: syncExtremes,
        afterSetExtremes: afterSetExtremes
      },
      labels: {
        rotation: -45,
        style: {
          fontSize: '10px'
        }
      }
    },
    yAxis: {
      title: { text: `${title} (${unit})` },
    },
    tooltip: {
      shared: true,
      crosshairs: true,
      positioner: () => ({ x: 80, y: 50 }),
    },
    series: [
      {
        name: title,
        data: values.value || [],
        color,
        showInNavigator: true,
      },
    ],
    navigator: { enabled: true },
    scrollbar: { enabled: true },
    credits: { enabled: false },
    legend: { enabled: false },
  }
}

// Get configuration options for combined chart
function getCombinedChartOptions() {
  return {
    chart: {
      renderTo: 'combinedChart',
      type: 'line',
      zoomType: 'x',
      panning: true,
      panKey: 'shift',
      resetZoomButton: {
        position: {
          align: 'right',
          x: -10,
          y: 10
        },
        theme: {
          fill: 'white',
          stroke: '#999',
          r: 5,
          style: {
            color: '#003399'
          }
        }
      }
    },
    title: {
      text: 'Combined Sensor Data',
    },
    xAxis: {
      categories: timeCategories.value,
      crosshair: true,
      events: {
        setExtremes: syncExtremes,
        afterSetExtremes: afterSetExtremes
      },
      labels: {
        rotation: -45,
        style: {
          fontSize: '10px'
        }
      }
    },
    yAxis: [
      {
        title: {
          text: 'Temperature (°C) & Humidity (%)',
        },
        labels: {
          format: '{value}',
        },
        opposite: false,
      },
      {
        title: {
          text: 'Pressure (hPa)',
        },
        labels: {
          format: '{value}',
        },
        opposite: true,
      },
    ],
    tooltip: {
      shared: true,
      crosshairs: true,
    },
    series: [
      {
        name: 'Temperature',
        data: temperatureData.value || [],
        color: sensorData.temperature.color,
        yAxis: 0,
      },
      {
        name: 'Humidity',
        data: humidityData.value || [],
        color: sensorData.humidity.color,
        yAxis: 0,
      },
      {
        name: 'Pressure',
        data: pressureData.value || [],
        color: sensorData.pressure.color,
        yAxis: 1,
      },
    ],
    legend: {
      layout: 'horizontal',
      align: 'center',
      verticalAlign: 'bottom',
    },
    navigator: {
      enabled: true,
      height: 20
    },
    scrollbar: {
      enabled: true
    },
    credits: {
      enabled: false,
    },
    plotOptions: {
      series: {
        marker: {
          enabled: false,
          states: {
            hover: {
              enabled: true,
              radius: 3
            }
          }
        }
      }
    }
  }
}

// =============================================
// CHART MANAGEMENT FUNCTIONS
// =============================================

// Update charts with new data
function updateCharts() {
  if (charts.length) {
    charts.forEach((chart, index) => {
      if (chart && chart.series && chart.series[0]) {
        if (index === 0) chart.series[0].setData(temperatureData.value)
        if (index === 1) chart.series[0].setData(humidityData.value)
        if (index === 2) chart.series[0].setData(pressureData.value)
        chart.xAxis[0].setCategories(timeCategories.value)
      }
    })
  }

  if (combinedChart && combinedChart.series) {
    combinedChart.series[0].setData(temperatureData.value)
    combinedChart.series[1].setData(humidityData.value)
    combinedChart.series[2].setData(pressureData.value)
    combinedChart.xAxis[0].setCategories(timeCategories.value)
  }
}

// =============================================
// CHART SYNCHRONIZATION FUNCTIONS
// =============================================

// Sync tooltip across charts
function syncTooltip(e) {
  charts.forEach((chart) => {
    if (!chart?.series[0]) return
    const point = chart.series[0].searchPoint(e, true)
    if (point) {
      point.onMouseOver()
      chart.tooltip.refresh(point)
      chart.xAxis[0].drawCrosshair(e, point)
    }
  })
}

// Sync zoom extremes across charts
function syncExtremes(e) {
  // Don't sync if this is part of a syncing operation already
  if (e.trigger === 'syncExtremes') {
    return;
  }

  // For regular extremes syncing
  const allCharts = [...charts];
  if (combinedChart) allCharts.push(combinedChart);

  allCharts.forEach((chart) => {
    if (chart !== this.chart && chart.xAxis[0].setExtremes) {
      chart.xAxis[0].setExtremes(e.min, e.max, undefined, false, { trigger: 'syncExtremes' });
    }
  });
}

// Sync reset zoom across all charts
function syncResetZoom() {
  if (isResetting) return;

  isResetting = true;

  // Reset all individual charts
  charts.forEach(chart => {
    if (chart && chart.xAxis && chart.xAxis[0]) {
      chart.xAxis[0].setExtremes(null, null);
    }
  });

  // Reset combined chart
  if (combinedChart && combinedChart.xAxis && combinedChart.xAxis[0]) {
    combinedChart.xAxis[0].setExtremes(null, null);
  }

  isResetting = false;
}

// Handle after extremes are set - detect reset zoom
function afterSetExtremes(e) {
  // If this is a reset (min/max are null or undefined), sync all charts
  if (e.min === undefined && e.max === undefined && !isResetting) {
    syncResetZoom();
  }
}

// =============================================
// LIFECYCLE HOOKS
// =============================================
// Component mounted - Initialize everything
onMounted(() => {
  // =============================================
  // FIRESTORE DATA SUBSCRIPTION
  // =============================================

  // Fetch real data from Firestore, ordered by timestamp
  const q = query(
    collection(db, 'sensor_readings'),
    orderBy('datetime', 'asc'), // Order by datetime field ascending
  )

  unsubscribeFirestore = onSnapshot(q, (querySnapshot) => {
    const realSensorData = []
    querySnapshot.forEach((doc) => {
      realSensorData.push(doc.data())
    })

    console.log('Current data in db: ', realSensorData)

    // Store all data
    allSensorData.value = realSensorData

    // Process data based on filter state
    if (!isFilterActive.value) {
      processAndDisplayData(realSensorData)
    } else {
      processAndDisplayData(filteredSensorData.value)
    }
  })

  // =============================================
  // CHART INITIALIZATION
  // =============================================

  // Initialize individual charts
  charts = [
    ['tempChart', 'Temperature', sensorData.temperature],
    ['humidityChart', 'Humidity', sensorData.humidity],
    ['pressureChart', 'Pressure', sensorData.pressure],
  ].map(([id, title, data]) => Highcharts.chart(getChartOptions(id, title, data)))

  // Initialize combined chart
  combinedChart = Highcharts.chart(getCombinedChartOptions())

  // =============================================
  // EVENT HANDLERS SETUP
  // =============================================

  const chartIds = ['tempChart', 'humidityChart', 'pressureChart']
  const moveEvents = ['mousemove', 'touchmove', 'touchstart']

  // Add mouse/touch move events for tooltip synchronization
  moveEvents.forEach((eventType) => {
    chartIds.forEach((id, idx) => {
      const el = document.getElementById(id)
      if (el) {
        const handler = (e) => syncTooltip(charts[idx].pointer.normalize(e))
        el.addEventListener(eventType, handler)
        eventListeners.push({ el, eventType, handler })
      }
    })
  })

  // Add mouse leave handlers to hide tooltips
  chartIds.forEach((id) => {
    const el = document.getElementById(id)
    if (el) {
      const handler = () =>
        charts.forEach((chart) => {
          chart.tooltip.hide()
          chart.xAxis[0].hideCrosshair()
        })
      el.addEventListener('mouseleave', handler)
      eventListeners.push({ el, eventType: 'mouseleave', handler })
    }
  })
})

// Component unmounted - Clean up everything
onBeforeUnmount(() => {
  // Clean up Firestore listener
  if (unsubscribeFirestore) {
    unsubscribeFirestore()
  }

  // Clean up event listeners
  eventListeners.forEach(({ el, eventType, handler }) =>
    el.removeEventListener(eventType, handler)
  )

  // Destroy charts
  charts.forEach((chart) => chart?.destroy?.())
  if (combinedChart) combinedChart.destroy()

  // Reset variables
  eventListeners = []
  charts = []
  combinedChart = null
})
</script>

<template>
  <main class="min-h-screen bg-gray-50 font-inter leading-relaxed tracking-wide">
    <div class="container mx-auto p-6 max-w-7xl">
      <div class="flex justify-between items-center mb-16">
        <div>
          <h1 class="text-4xl font-light text-gray-800 tracking-wide">Food Monitor</h1>
        </div>

        <div class="flex gap-3">
          <!-- Export CSV Button -->
          <button
            class="bg-gray-600 hover:bg-gray-700 text-white font-medium py-2.5 px-5 rounded-md transition-all duration-200 flex items-center justify-center shadow-sm border border-gray-300"
            @click="exportToCSV"
          >
            <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 mr-2" fill="none" viewBox="0 0 24 24" stroke="currentColor">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-4l-4 4m0 0l-4-4m4 4V4" />
            </svg>
            Export
          </button>

          <!-- View Images Button -->
          <router-link to='/images'
            class="nav-link bg-gray-800 hover:bg-gray-900 text-white font-medium py-2.5 px-5 rounded-md transition-all duration-200 flex items-center justify-center shadow-sm">
            <svg
              xmlns="http://www.w3.org/2000/svg"
              class="h-4 w-4 mr-2"
              viewBox="0 0 20 20"
              fill="currentColor"
            >
              <path
                fill-rule="evenodd"
                d="M4 5a2 2 0 00-2 2v8a2 2 0 002 2h12a2 2 0 002-2V7a2 2 0 00-2-2h-1.586a1 1 0 01-.707-.293l-1.121-1.121A2 2 0 0011.172 3H8.828a2 2 0 00-1.414.586L6.293 4.707A1 1 0 015.586 5H4zm6 9a3 3 0 100-6 3 3 0 000 6z"
                clip-rule="evenodd"
              />
            </svg>
            View Images
          </router-link>
        </div>
      </div>

      <!-- Additional spacer for gap between title and content -->
      <div class="h-4"></div>

      <!-- Current Data Section -->
      <section>
        <div class="bg-white shadow-sm rounded-lg p-8 border border-gray-200">
          <div class="mb-6">
            <h2 class="text-xl font-medium text-gray-800 mb-1">Current Readings</h2>
            <p class="text-sm text-gray-500">{{ currentDate }} • {{ currentTimeOnly }}</p>
          </div>

          <div class="grid grid-cols-1 md:grid-cols-3 gap-8">
            <!-- Current Temperature -->
            <div class="text-center">
              <div class="text-3xl font-light text-red-400 mb-1">{{ currentTemperature }}°C</div>
              <div class="text-sm text-gray-600 font-medium">Temperature</div>
            </div>

            <!-- Current Humidity -->
            <div class="text-center">
              <div class="text-3xl font-light text-blue-400 mb-1">{{ currentHumidity }}%</div>
              <div class="text-sm text-gray-600 font-medium">Humidity</div>
            </div>

            <!-- Current Pressure -->
            <div class="text-center">
              <div class="text-3xl font-light text-green-400 mb-1">{{ currentPressure }} hPa</div>
              <div class="text-sm text-gray-600 font-medium">Pressure</div>
            </div>
          </div>
        </div>
      </section>

      <!-- Spacer -->
      <div class="h-8"></div>

      <!-- Date Range Filter Section -->
      <section>
        <div class="bg-white shadow-sm rounded-lg p-8 border border-gray-200">
          <h2 class="text-xl font-medium text-gray-800 mb-6">Date Range</h2>
          <div class="flex flex-wrap items-end gap-4">
            <div>
              <label for="startDate" class="block text-sm font-medium text-gray-600 mb-2">
                Start Date
              </label>
              <input
                type="date"
                id="startDate"
                v-model="startDate"
                class="border border-gray-300 rounded-md px-4 py-2.5 text-sm focus:outline-none focus:ring-2 focus:ring-gray-400 focus:border-transparent"
              />
            </div>
            <div>
              <label for="endDate" class="block text-sm font-medium text-gray-600 mb-2">
                End Date
              </label>
              <input
                type="date"
                id="endDate"
                v-model="endDate"
                class="border border-gray-300 rounded-md px-4 py-2.5 text-sm focus:outline-none focus:ring-2 focus:ring-gray-400 focus:border-transparent"
              />
            </div>
            <div class="flex gap-3">
              <button
                @click="applyDateFilter"
                class="bg-gray-800 hover:bg-gray-900 text-white font-medium py-2.5 px-5 rounded-md transition-all duration-200"
              >
                Apply
              </button>
              <button
                @click="resetFilters"
                class="bg-gray-600 hover:bg-gray-700 text-white font-medium py-2.5 px-5 rounded-md transition-all duration-200"
              >
                Reset
              </button>
            </div>

            <div v-if="isFilterActive" class="ml-4 py-2 px-3 bg-gray-100 text-gray-700 rounded-md text-sm flex items-center border border-gray-200">
              <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 mr-1" viewBox="0 0 20 20" fill="currentColor">
                <path fill-rule="evenodd" d="M3 3a1 1 0 011-1h12a1 1 0 011 1v3a1 1 0 01-.293.707L12 11.414V15a1 1 0 01-.293.707l-2 2A1 1 0 018 17v-5.586L3.293 6.707A1 1 0 013 6V3z" clip-rule="evenodd" />
              </svg>
              Filter Active
            </div>
          </div>
        </div>
      </section>

      <!-- Spacer -->
      <div class="h-8"></div>

      <!-- Historical Data Charts Section -->
      <section>
        <div class="bg-white shadow-sm rounded-lg p-8 border border-gray-200">
          <h2 class="text-xl font-medium text-gray-800 mb-8">Historical Data</h2>
          <div class="grid grid-cols-1 md:grid-cols-3 gap-8">
            <div id="tempChart"></div>
            <div id="humidityChart"></div>
            <div id="pressureChart"></div>
          </div>
        </div>
      </section>

      <!-- Spacer -->
      <div class="h-8"></div>

      <!-- Combined Chart Section -->
      <section>
        <div class="bg-white shadow-sm rounded-lg p-8 border border-gray-200">
          <h2 class="text-xl font-medium text-gray-800 mb-8">Combined Historical Data</h2>
          <div id="combinedChart" class="w-full h-96"></div>
        </div>
      </section>
    </div>
  </main>
</template>

<style scoped>
/* Font styling for modern look */
.font-inter {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
}

/* Chart containers */
#tempChart,
#humidityChart,
#pressureChart,
#combinedChart {
  min-height: 300px;
}

/* Ensure proper chart sizing on mobile */
@media (max-width: 768px) {
  #tempChart,
  #humidityChart,
  #pressureChart {
    min-height: 250px;
  }
}

/* Custom focus styles for better accessibility */
input:focus,
button:focus {
  box-shadow: 0 0 0 2px rgba(156, 163, 175, 0.5);
}

/* Smooth transitions for interactive elements */
button {
  transition: all 0.2s ease-in-out;
}

/* Custom scrollbar for better visual consistency */
::-webkit-scrollbar {
  width: 6px;
  height: 6px;
}

::-webkit-scrollbar-track {
  background: #f1f5f9;
}

::-webkit-scrollbar-thumb {
  background: #cbd5e1;
  border-radius: 3px;
}

::-webkit-scrollbar-thumb:hover {
  background: #94a3b8;
}
</style>
