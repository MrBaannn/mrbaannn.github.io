<template>
  <div class="nav-chart" ref="chartWrapper">
    <canvas ref="canvas" @mousemove="onMouseMove" @mouseleave="onMouseLeave"></canvas>
    <div
      v-if="tooltip.show"
      class="chart-tooltip"
      :style="{ left: tooltip.x + 'px', top: tooltip.y + 'px' }"
    >
      <div class="tooltip-date">{{ tooltip.date }}</div>
      <div class="tooltip-nav">{{ tooltip.pct }}</div>
    </div>
  </div>
</template>

<script setup>
import { ref, watch, onMounted, onUnmounted, nextTick } from 'vue'

const props = defineProps({
  data: { type: Array, default: () => [] }
})

const canvas = ref(null)
const chartWrapper = ref(null)
const tooltip = ref({ show: false, x: 0, y: 0, date: '', pct: '' })

let resizeObserver = null
let pointPositions = []

const COLORS = {
  line: '#5b8a72',
  lineLight: 'rgba(91, 138, 114, 0.08)',
  grid: '#f0f2f5',
  text: '#9ba8b5',
  dot: '#5b8a72',
  dotHover: '#3a6a52',
  baseline: '#e0e4e8'
}

// Allowed base intervals for Y-axis
const ALLOWED_BASES = [0.05, 0.10, 0.15, 0.20, 0.25, 0.30, 0.40, 0.50, 0.60, 0.70, 0.80]

function pickInterval(range) {
  // We want roughly 4-6 grid intervals to cover the range
  const targetSteps = 5
  const rawInterval = range / targetSteps
  if (rawInterval <= 0) return 0.05

  // Find the best allowed interval (base * 10^N, N >= 0)
  let best = null
  let bestDiff = Infinity
  for (const base of ALLOWED_BASES) {
    // Try multipliers: 1, 10, 100, 1000, ...
    let multiplier = 1
    for (let n = 0; n < 6; n++) {
      const candidate = base * multiplier
      const diff = Math.abs(candidate - rawInterval)
      if (diff < bestDiff) {
        bestDiff = diff
        best = candidate
      }
      multiplier *= 10
      if (candidate > range * 2) break
    }
  }
  return best || 0.05
}

function drawChart() {
  const cvs = canvas.value
  if (!cvs || !chartWrapper.value) return

  const wrapper = chartWrapper.value
  const dpr = window.devicePixelRatio || 1
  const width = wrapper.clientWidth
  const height = wrapper.clientHeight || 280

  cvs.width = width * dpr
  cvs.height = height * dpr
  cvs.style.width = width + 'px'
  cvs.style.height = height + 'px'

  const ctx = cvs.getContext('2d')
  ctx.scale(dpr, dpr)
  ctx.clearRect(0, 0, width, height)

  const data = props.data
  if (!data || data.length === 0) {
    ctx.fillStyle = COLORS.text
    ctx.font = '14px "Noto Sans SC", sans-serif'
    ctx.textAlign = 'center'
    ctx.fillText('暂无数据', width / 2, height / 2)
    return
  }

  if (data.length <= 1) {
    ctx.fillStyle = COLORS.text
    ctx.font = '14px "Noto Sans SC", sans-serif'
    ctx.textAlign = 'center'
    ctx.fillText('数据缺失', width / 2, height / 2)
    return
  }

  const allSameNav = data.every(d => d.nav === data[0].nav)
  if (allSameNav) {
    ctx.fillStyle = COLORS.text
    ctx.font = '14px "Noto Sans SC", sans-serif'
    ctx.textAlign = 'center'
    ctx.fillText('净值无变化', width / 2, height / 2)
    return
  }

  const padding = { top: 24, right: 24, bottom: 44, left: 64 }
  const chartW = width - padding.left - padding.right
  const chartH = height - padding.top - padding.bottom

  // Convert nav to percentage change from first point
  const firstNav = data[0].nav
  const pctValues = data.map(d => ((d.nav / firstNav) - 1) * 100)

  let minPct = Math.min(...pctValues)
  let maxPct = Math.max(...pctValues)

  // Pick a nice interval
  const rawRange = maxPct - minPct
  const interval = pickInterval(rawRange === 0 ? 0.1 : rawRange)

  // Compute grid bounds: snap to interval boundaries
  const gridMin = Math.floor(minPct / interval) * interval
  const gridMax = Math.ceil(maxPct / interval) * interval

  // Generate grid values
  const gridValues = []
  for (let v = gridMin; v <= gridMax + interval * 0.01; v += interval) {
    gridValues.push(parseFloat(v.toFixed(6)))
  }

  const yMin = gridValues[0]
  const yMax = gridValues[gridValues.length - 1]

  // Helper: data to pixel
  const xScale = (i) => padding.left + (data.length === 1 ? chartW / 2 : (i / (data.length - 1)) * chartW)
  const yScale = (v) => padding.top + chartH - ((v - yMin) / (yMax - yMin)) * chartH

  // Draw grid lines and Y-axis labels
  ctx.strokeStyle = COLORS.grid
  ctx.lineWidth = 1
  ctx.setLineDash([])
  for (const val of gridValues) {
    const y = yScale(val)
    ctx.beginPath()
    ctx.moveTo(padding.left, y)
    ctx.lineTo(width - padding.right, y)
    ctx.stroke()

    ctx.fillStyle = COLORS.text
    ctx.font = '11px "Noto Sans SC", sans-serif'
    ctx.textAlign = 'right'
    ctx.fillText(val.toFixed(2) + '%', padding.left - 8, y + 4)
  }

  // Draw baseline at 0% if in range
  if (yMin <= 0 && yMax >= 0) {
    const baseY = yScale(0)
    ctx.strokeStyle = COLORS.baseline
    ctx.lineWidth = 1
    ctx.setLineDash([4, 4])
    ctx.beginPath()
    ctx.moveTo(padding.left, baseY)
    ctx.lineTo(width - padding.right, baseY)
    ctx.stroke()
    ctx.setLineDash([])
  }

  // Draw area fill
  if (data.length > 1) {
    ctx.beginPath()
    ctx.moveTo(xScale(0), yScale(pctValues[0]))
    for (let i = 1; i < data.length; i++) {
      ctx.lineTo(xScale(i), yScale(pctValues[i]))
    }
    ctx.lineTo(xScale(data.length - 1), padding.top + chartH)
    ctx.lineTo(xScale(0), padding.top + chartH)
    ctx.closePath()

    const gradient = ctx.createLinearGradient(0, padding.top, 0, padding.top + chartH)
    gradient.addColorStop(0, 'rgba(91, 138, 114, 0.15)')
    gradient.addColorStop(1, 'rgba(91, 138, 114, 0.01)')
    ctx.fillStyle = gradient
    ctx.fill()
  }

  // Draw line
  ctx.strokeStyle = COLORS.line
  ctx.lineWidth = 2.5
  ctx.lineJoin = 'round'
  ctx.lineCap = 'round'
  ctx.beginPath()
  for (let i = 0; i < data.length; i++) {
    const x = xScale(i)
    const y = yScale(pctValues[i])
    if (i === 0) ctx.moveTo(x, y)
    else ctx.lineTo(x, y)
  }
  ctx.stroke()

  // Draw dots and store positions
  pointPositions = []
  for (let i = 0; i < data.length; i++) {
    const x = xScale(i)
    const y = yScale(pctValues[i])
    pointPositions.push({ x, y, date: data[i].date, pct: pctValues[i] })

    ctx.beginPath()
    ctx.arc(x, y, 4, 0, Math.PI * 2)
    ctx.fillStyle = '#fff'
    ctx.fill()
    ctx.strokeStyle = COLORS.dot
    ctx.lineWidth = 2
    ctx.stroke()
  }

  // X-axis labels (show at most ~8 labels to avoid crowding)
  const maxLabels = Math.min(data.length, 8)
  const step = Math.max(1, Math.floor(data.length / maxLabels))
  ctx.fillStyle = COLORS.text
  ctx.font = '11px "Noto Sans SC", sans-serif'
  ctx.textAlign = 'center'
  for (let i = 0; i < data.length; i += step) {
    const x = xScale(i)
    const dateStr = data[i].date.substring(5)
    ctx.fillText(dateStr, x, height - padding.bottom + 20)
  }
  // Always show last label
  if ((data.length - 1) % step !== 0 && data.length > 1) {
    const x = xScale(data.length - 1)
    const dateStr = data[data.length - 1].date.substring(5)
    ctx.fillText(dateStr, x, height - padding.bottom + 20)
  }
}

function onMouseMove(e) {
  const rect = canvas.value.getBoundingClientRect()
  const mx = e.clientX - rect.left
  const my = e.clientY - rect.top

  let closest = null
  let minDist = Infinity
  for (const p of pointPositions) {
    const dist = Math.sqrt((p.x - mx) ** 2 + (p.y - my) ** 2)
    if (dist < minDist && dist < 40) {
      minDist = dist
      closest = p
    }
  }

  if (closest) {
    const sign = closest.pct >= 0 ? '+' : ''
    tooltip.value = {
      show: true,
      x: closest.x,
      y: closest.y - 50,
      date: closest.date,
      pct: sign + closest.pct.toFixed(2) + '%'
    }
  } else {
    tooltip.value.show = false
  }
}

function onMouseLeave() {
  tooltip.value.show = false
}

onMounted(() => {
  nextTick(() => {
    drawChart()
    resizeObserver = new ResizeObserver(() => drawChart())
    if (chartWrapper.value) resizeObserver.observe(chartWrapper.value)
  })
})

onUnmounted(() => {
  if (resizeObserver) resizeObserver.disconnect()
})

watch(() => props.data, () => {
  nextTick(() => drawChart())
}, { deep: true })
</script>

<style scoped>
.nav-chart {
  position: relative;
  width: 100%;
  height: 100%;
  min-height: 280px;
}

canvas {
  display: block;
  width: 100%;
  height: 100%;
}

.chart-tooltip {
  position: absolute;
  background: rgba(44, 62, 80, 0.92);
  color: #fff;
  padding: 8px 12px;
  border-radius: 8px;
  font-size: 0.8rem;
  pointer-events: none;
  transform: translateX(-50%);
  white-space: nowrap;
  z-index: 10;
  box-shadow: 0 4px 12px rgba(0,0,0,0.15);
}

.tooltip-date {
  font-size: 0.7rem;
  opacity: 0.8;
  margin-bottom: 2px;
}

.tooltip-nav {
  font-weight: 600;
  font-variant-numeric: tabular-nums;
}
</style>
