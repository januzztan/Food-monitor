<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue'
import Highcharts from 'highcharts'
import { db } from '@/firebase'
import { collection, query, onSnapshot, orderBy } from 'firebase/firestore'

// Set Highcharts global options
Highcharts.setOptions({
  lang: {
    decimalPoint: '.',
    thousandsSep: ',',
  },
})

// Function to take a picture
const takePicture = () => {
  console.log('Taking picture...')
}

// Current sensor values (reactive)
const currentTemperature = ref(0)
const currentHumidity = ref(0)
const currentPressure = ref(0)
const currentTime = ref('--:--')

// Chart data (reactive)
const timeCategories = ref([])
const temperatureData = ref([])
const humidityData = ref([])
const pressureData = ref([])

// Sensor Data
const sensorData = {
  temperature: {
    data: temperatureData,
    color: 'rgb(255, 99, 132)',
    unit: '°C',
  },
  humidity: {
    data: humidityData,
    color: 'rgb(54, 162, 235)',
    unit: '%',
  },
  pressure: {
    data: pressureData,
    color: 'rgb(75, 192, 192)',
    unit: 'hPa',
  },
}

// Chart Management
let charts = []
let combinedChart = null
let eventListeners = []
let unsubscribeFirestore = null
// Track if we're currently in a reset operation to prevent infinite loops
let isResetting = false;

// Function to sync reset zoom across all charts
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

function getCombinedChartOptions() {
  return {
    chart: {
      renderTo: 'combinedChart',
      type: 'line',
      zoomType: 'x',  // Add zoom capability on x axis
      panning: true,  // Enable panning
      panKey: 'shift', // Pan when shift key is pressed
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
        setExtremes: syncExtremes,  // Sync with other charts
        afterSetExtremes: afterSetExtremes
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
          enabled: false, // Disable markers for cleaner look when zoomed
          states: {
            hover: {
              enabled: true, // But show on hover
              radius: 3
            }
          }
        }
      }
    }
  }
}

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

// Get latest reading for current display
function updateCurrentReadings(data) {
  if (!data || !data.length) return

  // Get the last element (most recent) if ordered by datetime ascending
  const latest = data[data.length - 1]

  // Update current values
  currentTemperature.value = latest.temperature?.toFixed(1) || 0
  currentHumidity.value = latest.humidity?.toFixed(1) || 0
  currentPressure.value = latest.pressure?.toFixed(1) || 0
  currentTime.value = latest.timestamp?.split(' ')[1] || '--:--'
}

// Chart Synchronization
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

// Lifecycle Hooks
onMounted(() => {
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

    if (realSensorData.length) {
      // Since data is already ordered by datetime from Firestore
      // No need for additional sorting in the client side

      // Extract and prepare data for charts
      timeCategories.value = realSensorData.map((reading) => {
        // Extract time part from timestamp (format: 22/05/2025 14:11:10)
        return reading.timestamp ? reading.timestamp.split(' ')[1].substring(0, 5) : ''
      })

      temperatureData.value = realSensorData.map((reading) => reading.temperature || 0)
      humidityData.value = realSensorData.map((reading) => reading.humidity || 0)
      pressureData.value = realSensorData.map((reading) => reading.pressure || 0)

      // Update current readings with latest data (still need to find the most recent)
      updateCurrentReadings(realSensorData)

      // Update charts if they exist
      updateCharts()
    }
  })

  // Initialize charts with empty data
  charts = [
    ['tempChart', 'Temperature', sensorData.temperature],
    ['humidityChart', 'Humidity', sensorData.humidity],
    ['pressureChart', 'Pressure', sensorData.pressure],
  ].map(([id, title, data]) => Highcharts.chart(getChartOptions(id, title, data)))

  combinedChart = Highcharts.chart(getCombinedChartOptions())

  // Event Handlers
  const chartIds = ['tempChart', 'humidityChart', 'pressureChart']
  const moveEvents = ['mousemove', 'touchmove', 'touchstart']

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

  // Mouse Leave Handler
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

onBeforeUnmount(() => {
  // Clean up Firestore listener
  if (unsubscribeFirestore) {
    unsubscribeFirestore()
  }

  // Clean up event listeners
  eventListeners.forEach(({ el, eventType, handler }) => el.removeEventListener(eventType, handler))

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
  <main class="font-sans leading-normal tracking-normal">
    <div class="container mx-auto p-4">
      <div class="flex justify-between items-center mb-12">
        <div>
          <p class="text-5xl font-bold pb-5">Food Monitor</p>
        </div>

        <button
          class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-lg transition-colors duration-200 flex items-center justify-center"
          @click="takePicture"
        >
          <svg
            xmlns="http://www.w3.org/2000/svg"
            class="h-5 w-5 mr-2"
            viewBox="0 0 20 20"
            fill="currentColor"
          >
            <path
              fill-rule="evenodd"
              d="M4 5a2 2 0 00-2 2v8a2 2 0 002 2h12a2 2 0 002-2V7a2 2 0 00-2-2h-1.586a1 1 0 01-.707-.293l-1.121-1.121A2 2 0 0011.172 3H8.828a2 2 0 00-1.414.586L6.293 4.707A1 1 0 015.586 5H4zm6 9a3 3 0 100-6 3 3 0 000 6z"
              clip-rule="evenodd"
            />
          </svg>
          Take Picture
        </button>
      </div>

      <!-- Current Data Section -->
      <section>
        <div class="bg-white shadow-lg rounded-lg p-6 border-t-4 border-blue-500">
          <p class="text-2xl font-bold mt-2 pb-5">Current Data</p>

          <div class="grid grid-cols-1 md:grid-cols-4 gap-6">
            <!-- Current Time -->
            <div class="bg-white shadow-md rounded-lg p-4 text-center">
              <h2 class="text-xl font-semibold mb-2">Time</h2>
              <p class="text-2xl font-bold text-gray-700">{{ currentTime }}</p>
            </div>

            <!-- Current Temperature -->
            <div class="bg-white shadow-md rounded-lg p-4 text-center">
              <h2 class="text-xl font-semibold mb-2">Temperature</h2>
              <p class="text-2xl font-bold text-red-500">{{ currentTemperature }}°C</p>
            </div>

            <!-- Current Humidity -->
            <div class="bg-white shadow-md rounded-lg p-4 text-center">
              <h2 class="text-xl font-semibold mb-2">Humidity</h2>
              <p class="text-2xl font-bold text-blue-500">{{ currentHumidity }}%</p>
            </div>

            <!-- Current Pressure -->
            <div class="bg-white shadow-md rounded-lg p-4 text-center">
              <h2 class="text-xl font-semibold mb-2">Pressure</h2>
              <p class="text-2xl font-bold text-green-500">{{ currentPressure }} hPa</p>
            </div>
          </div>
        </div>
      </section>

      <!-- Spacer -->
      <div class="h-5"></div>

      <!-- Historical Data Charts Section -->
      <section>
        <div class="bg-white shadow-lg rounded-lg p-6 border-t-4 border-green-500">
          <h2 class="text-2xl font-bold mb-6">Historical Data</h2>
          <div class="grid grid-cols-1 md:grid-cols-3 gap-6">
            <div id="tempChart"></div>
            <div id="humidityChart"></div>
            <div id="pressureChart"></div>
          </div>
        </div>
      </section>

      <!-- Spacer -->
      <div class="h-5"></div>

      <!-- Combined Chart Section -->
      <section>
        <div class="bg-white shadow-lg rounded-lg p-6 border-t-4 border-purple-500">
          <h2 class="text-2xl font-bold mb-6">Combined Historical Data</h2>
          <div id="combinedChart" class="w-full h-96"></div>
        </div>
      </section>
    </div>
  </main>
</template>
