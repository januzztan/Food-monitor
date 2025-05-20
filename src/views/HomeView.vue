<script setup>
import { onMounted, onBeforeUnmount } from 'vue'
import Highcharts from 'highcharts'

// Set Highcharts global options
Highcharts.setOptions({
  lang: {
    decimalPoint: '.',
    thousandsSep: ',',
  },
})

// Function to take a picture
const takePicture = () => {
  // Add your camera functionality here
  console.log('Taking picture...')
}

// Data Configuration
const categories = Array.from({ length: 61 }, (_, i) => {
  const hour = Math.floor(i / 60) + 10
  const minute = i % 60
  return `${hour}:${minute.toString().padStart(2, '0')}`
})

// Sensor Data
const sensorData = {
  temperature: {
    data: Array(61)
      .fill(22)
      .map((val, i) => Math.floor(i / 10) + val),
    color: 'rgb(255, 99, 132)',
    unit: '°C',
  },
  humidity: {
    data: Array(61)
      .fill(60)
      .map((val, i) => Math.floor(i / 10) + val),
    color: 'rgb(54, 162, 235)',
    unit: '%',
  },
  pressure: {
    data: Array(61)
      .fill(1012)
      .map((val, i) => val + (i > 30 ? 2 : 0)),
    color: 'rgb(75, 192, 192)',
    unit: 'hPa',
  },
}

// Chart Management
let charts = []
let eventListeners = []

function getChartOptions(id, title, data) {
  const { color, unit, data: values } = data

  return {
    chart: {
      renderTo: id,
      type: 'line',
      zoomType: 'x',
      events: { selection: syncExtremes },
    },
    title: { text: title },
    xAxis: {
      categories,
      crosshair: true,
      events: { setExtremes: syncExtremes },
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
        data: values,
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
  if (e.trigger !== 'syncExtremes') {
    charts.forEach((chart) => {
      if (chart !== this.chart && chart.xAxis[0].setExtremes) {
        chart.xAxis[0].setExtremes(e.min, e.max, undefined, false, { trigger: 'syncExtremes' })
      }
    })
  }
}

// Lifecycle Hooks
onMounted(() => {
  charts = [
    ['tempChart', 'Temperature', sensorData.temperature],
    ['humidityChart', 'Humidity', sensorData.humidity],
    ['pressureChart', 'Pressure', sensorData.pressure],
  ].map(([id, title, data]) => Highcharts.chart(getChartOptions(id, title, data)))

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
  eventListeners.forEach(({ el, eventType, handler }) => el.removeEventListener(eventType, handler))
  charts.forEach((chart) => chart?.destroy?.())
  eventListeners = []
  charts = []
})
</script>

<template>
  <main class="font-sans leading-normal tracking-normal">
    <div class="container mx-auto p-2">
      <div class="flex justify-between items-center mb-5">
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

      <div class="space-y-8">
        <div class="bg-white shadow-md rounded-lg p-4">
          <p class="text-2xl font-bold mt-2 pb-5">Current Data</p>

          <div class="grid grid-cols-1 md:grid-cols-4 gap-6">
            <!-- Current Time -->
            <div class="bg-white shadow-md rounded-lg p-4 text-center">
              <h2 class="text-xl font-semibold mb-2">Time</h2>
              <p class="text-2xl font-bold text-gray-700">14:00</p>
            </div>

            <!-- Current Temperature -->
            <div class="bg-white shadow-md rounded-lg p-4 text-center">
              <h2 class="text-xl font-semibold mb-2">Temperature</h2>
              <p class="text-2xl font-bold text-red-500">26°C</p>
            </div>

            <!-- Current Humidity -->
            <div class="bg-white shadow-md rounded-lg p-4 text-center">
              <h2 class="text-xl font-semibold mb-2">Humidity</h2>
              <p class="text-2xl font-bold text-blue-500">64%</p>
            </div>

            <!-- Current Pressure -->
            <div class="bg-white shadow-md rounded-lg p-4 text-center">
              <h2 class="text-xl font-semibold mb-2">Pressure</h2>
              <p class="text-2xl font-bold text-green-500">1012 hPa</p>
            </div>
          </div>
        </div>

        <!-- Historical Data Charts (Stacked) -->
        <div class="bg-white shadow-md rounded-lg p-4">
          <h2 class="text-2xl font-bold mb-4">Historical Data</h2>
          <div id="tempChart" class="mb-6"></div>
          <div id="humidityChart" class="mb-6"></div>
          <div id="pressureChart"></div>
        </div>
      </div>
    </div>
  </main>
</template>
