<script setup>
import {
  CategoryScale,
  Chart as ChartJS,
  Filler,
  LineElement,
  LinearScale,
  PointElement,
  TimeScale,
  Tooltip,
} from 'chart.js'
import 'chartjs-adapter-date-fns'
import { computed } from 'vue'
import { Line } from 'vue-chartjs'

ChartJS.register(CategoryScale, LinearScale, PointElement, LineElement, Tooltip, Filler, TimeScale)

const props = defineProps({
  history: {
    type: Array,
    default: () => [],
  },
  formatPrice: {
    type: Function,
    required: true,
  },
  formatShortDate: {
    type: Function,
    required: true,
  },
})

const chartData = computed(() => ({
  datasets: [
    {
      data: props.history.map((point) => ({
        x: new Date(point.recorded_at),
        y: point.price,
      })),
      borderColor: '#72f5a2',
      backgroundColor: 'rgba(114, 245, 162, 0.18)',
      borderWidth: 3,
      tension: 0.35,
      fill: true,
      pointRadius: 0,
      pointHoverRadius: 4,
      pointHoverBackgroundColor: '#72f5a2',
      pointHoverBorderColor: '#0a1220',
      pointHoverBorderWidth: 2,
    },
  ],
}))

const chartOptions = computed(() => ({
  responsive: true,
  maintainAspectRatio: false,
  animation: false,
  normalized: true,
  interaction: {
    intersect: false,
    mode: 'index',
  },
  plugins: {
    legend: {
      display: false,
    },
    tooltip: {
      displayColors: false,
      backgroundColor: '#101a29',
      borderColor: 'rgba(114, 245, 162, 0.24)',
      borderWidth: 1,
      padding: 10,
      titleColor: '#eef3fb',
      bodyColor: '#eef3fb',
      callbacks: {
        title(items) {
          const point = items[0]?.raw
          return point ? props.formatShortDate(point.x) : ''
        },
        label(context) {
          return props.formatPrice(context.raw?.y ?? 0)
        },
      },
    },
  },
  scales: {
    x: {
      type: 'time',
      display: false,
    },
    y: {
      display: false,
    },
  },
  elements: {
    line: {
      capBezierPoints: true,
    },
  },
}))
</script>

<template>
  <Line :data="chartData" :options="chartOptions" />
</template>
