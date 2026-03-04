<template>
  <div class="holdings-pie" ref="pieWrapper">
    <div class="pie-layout">
      <div class="pie-canvas-wrap">
        <canvas
          ref="canvas"
          @mousemove="onMouseMove"
          @mouseleave="onMouseLeave"
        ></canvas>
      </div>
      <!-- Legend on the right -->
      <div class="pie-legend">
        <div
          v-for="(cat, i) in validCategories"
          :key="cat.name"
          class="legend-item"
          :class="{ active: hoveredIndex === i }"
          @mouseenter="hoveredIndex = i; drawPie()"
          @mouseleave="hoveredIndex = -1; drawPie()"
        >
          <span class="legend-dot" :style="{ background: categoryColors[i] }"></span>
          <span class="legend-name">{{ cat.name }}</span>
          <span class="legend-pct">{{ cat.ratio_pct.toFixed(2) }}%</span>
        </div>
      </div>
    </div>
    <!-- Detail tooltip -->
    <div
      v-if="hoveredIndex >= 0 && hoveredDetail"
      class="pie-detail"
      :style="{ left: detailPos.x + 'px', top: detailPos.y + 'px' }"
    >
      <div class="detail-title">{{ hoveredDetail.name }}</div>
      <div class="detail-total">{{ hoveredDetail.ratio_pct.toFixed(2) }}%</div>
      <div v-if="sortedItems.length > 0" class="detail-items">
        <div v-for="item in sortedItems" :key="item.name" class="detail-item">
          <span class="detail-item-name">{{ item.name }}</span>
          <span class="detail-item-pct">{{ item.pct.toFixed(2) }}%</span>
        </div>
      </div>
      <div v-else class="detail-empty">无细分项</div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted, onUnmounted, nextTick } from 'vue'

const props = defineProps({
  categories: { type: Array, default: () => [] }
})

const canvas = ref(null)
const pieWrapper = ref(null)
const hoveredIndex = ref(-1)
const detailPos = ref({ x: 0, y: 0 })

let resizeObserver = null
let sliceAngles = [] // Store slice start/end angles for hit testing

// Fixed color map for the 9 categories (order matches JSON)
const CATEGORY_COLOR_MAP = {
  '零钱':     '#b0b8c4', // 灰色
  '固收':     '#7bc8a4', // 浅绿色
  '固收+':    '#a8d86e', // 黄绿色
  '黄金':     '#f0c75e', // 金黄色
  'A股':      '#e06060', // 红色
  '港股':     '#d4699e', // 玫红色
  '美股':     '#4a90d9', // 海蓝色
  '全球市场':  '#6ec5e9', // 天蓝色
  '其它':     '#4a4a4a', // 黑色
}

const validCategories = computed(() => {
  return props.categories || []
})

const categoryColors = computed(() => {
  return validCategories.value.map(cat => CATEGORY_COLOR_MAP[cat.name] || '#999')
})

const hoveredDetail = computed(() => {
  if (hoveredIndex.value >= 0 && hoveredIndex.value < validCategories.value.length) {
    return validCategories.value[hoveredIndex.value]
  }
  return null
})

// Sorted items: by pct descending, but "其它" always last
const sortedItems = computed(() => {
  if (!hoveredDetail.value || !hoveredDetail.value.items) return []
  const items = Object.entries(hoveredDetail.value.items).map(([name, pct]) => ({ name, pct }))
  if (items.length === 0) return []

  // Separate "其它" from the rest
  const others = items.filter(i => i.name === '其它')
  const rest = items.filter(i => i.name !== '其它')
  rest.sort((a, b) => b.pct - a.pct)
  return [...rest, ...others]
})

// Determine if a hex color is light (needs dark text)
function isLightColor(hex) {
  const r = parseInt(hex.slice(1, 3), 16)
  const g = parseInt(hex.slice(3, 5), 16)
  const b = parseInt(hex.slice(5, 7), 16)
  const luminance = (0.299 * r + 0.587 * g + 0.114 * b) / 255
  return luminance > 0.6
}

function drawPie() {
  const cvs = canvas.value
  if (!cvs || !pieWrapper.value) return

  const dpr = window.devicePixelRatio || 1
  const canvasWrap = cvs.parentElement
  const size = Math.min(canvasWrap.clientWidth, canvasWrap.clientHeight, 400)
  cvs.width = size * dpr
  cvs.height = size * dpr
  cvs.style.width = size + 'px'
  cvs.style.height = size + 'px'

  const ctx = cvs.getContext('2d')
  ctx.scale(dpr, dpr)
  ctx.clearRect(0, 0, size, size)

  const cats = validCategories.value
  if (!cats.length) return

  const cx = size / 2
  const cy = size / 2
  const outerR = size / 2 - 16
  const innerR = outerR * 0.52 // Donut hole

  const total = cats.reduce((s, c) => s + c.ratio_pct, 0) || 1
  let startAngle = -Math.PI / 2

  sliceAngles = []

  for (let i = 0; i < cats.length; i++) {
    const sliceAngle = (cats[i].ratio_pct / total) * Math.PI * 2
    const endAngle = startAngle + sliceAngle

    const isHovered = hoveredIndex.value === i
    const offset = isHovered ? 8 : 0
    const midAngle = startAngle + sliceAngle / 2
    const ox = Math.cos(midAngle) * offset
    const oy = Math.sin(midAngle) * offset

    // Draw slice
    ctx.beginPath()
    ctx.arc(cx + ox, cy + oy, outerR, startAngle, endAngle)
    ctx.arc(cx + ox, cy + oy, innerR, endAngle, startAngle, true)
    ctx.closePath()
    ctx.fillStyle = categoryColors.value[i]
    if (isHovered) {
      ctx.shadowColor = 'rgba(0,0,0,0.15)'
      ctx.shadowBlur = 12
    } else {
      ctx.shadowColor = 'transparent'
      ctx.shadowBlur = 0
    }
    ctx.fill()
    ctx.shadowColor = 'transparent'
    ctx.shadowBlur = 0

    // White border between slices
    ctx.strokeStyle = '#fff'
    ctx.lineWidth = 2
    ctx.stroke()

    sliceAngles.push({ start: startAngle, end: endAngle, index: i })

    // Label on slice (only if big enough)
    if (cats[i].ratio_pct / total > 0.04) {
      const labelR = (outerR + innerR) / 2
      const lx = cx + ox + Math.cos(midAngle) * labelR
      const ly = cy + oy + Math.sin(midAngle) * labelR

      const textColor = isLightColor(categoryColors.value[i]) ? '#2c3e50' : '#fff'
      ctx.fillStyle = textColor
      ctx.font = `${isHovered ? '600' : '500'} 12px "Noto Sans SC", sans-serif`
      ctx.textAlign = 'center'
      ctx.textBaseline = 'middle'
      ctx.fillText(cats[i].name, lx, ly - 7)
      ctx.font = '11px "Noto Sans SC", sans-serif'
      ctx.fillText(cats[i].ratio_pct.toFixed(1) + '%', lx, ly + 8)
    }

    startAngle = endAngle
  }

  // Center text
  ctx.fillStyle = '#2c3e50'
  ctx.font = '600 15px "Noto Sans SC", sans-serif'
  ctx.textAlign = 'center'
  ctx.textBaseline = 'middle'
  ctx.fillText('持仓分布', cx, cy - 8)
  ctx.fillStyle = '#9ba8b5'
  ctx.font = '12px "Noto Sans SC", sans-serif'
  ctx.fillText(`共${cats.length}类`, cx, cy + 12)
}

function onMouseMove(e) {
  const rect = canvas.value.getBoundingClientRect()
  const mx = e.clientX - rect.left
  const my = e.clientY - rect.top
  const size = parseInt(canvas.value.style.width)
  const cx = size / 2
  const cy = size / 2
  const outerR = size / 2 - 16
  const innerR = outerR * 0.52

  const dx = mx - cx
  const dy = my - cy
  const dist = Math.sqrt(dx * dx + dy * dy)

  if (dist < innerR || dist > outerR) {
    if (hoveredIndex.value !== -1) {
      hoveredIndex.value = -1
      drawPie()
    }
    return
  }

  let angle = Math.atan2(dy, dx)
  // Normalize to match our start angle (-PI/2)
  if (angle < -Math.PI / 2) angle += Math.PI * 2

  let found = -1
  for (const s of sliceAngles) {
    let start = s.start
    let end = s.end
    if (start < -Math.PI / 2) start += Math.PI * 2
    if (end < -Math.PI / 2) end += Math.PI * 2
    if (angle >= start && angle < end) {
      found = s.index
      break
    }
  }

  if (found !== hoveredIndex.value) {
    hoveredIndex.value = found
    drawPie()
  }

  // Position detail tooltip
  if (found >= 0) {
    const wrapperRect = pieWrapper.value.getBoundingClientRect()
    detailPos.value = {
      x: e.clientX - wrapperRect.left + 16,
      y: e.clientY - wrapperRect.top - 10
    }
  }
}

function onMouseLeave() {
  hoveredIndex.value = -1
  drawPie()
}

onMounted(() => {
  nextTick(() => {
    drawPie()
    resizeObserver = new ResizeObserver(() => drawPie())
    if (pieWrapper.value) resizeObserver.observe(pieWrapper.value)
  })
})

onUnmounted(() => {
  if (resizeObserver) resizeObserver.disconnect()
})

watch(() => props.categories, () => {
  nextTick(() => drawPie())
}, { deep: true })
</script>

<style scoped>
.holdings-pie {
  position: relative;
}

.pie-layout {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 40px;
}

.pie-canvas-wrap {
  flex-shrink: 0;
  width: 360px;
  height: 360px;
  display: flex;
  align-items: center;
  justify-content: center;
}

canvas {
  cursor: pointer;
  display: block;
}

.pie-legend {
  display: flex;
  flex-direction: column;
  gap: 6px;
  min-width: 180px;
}

.legend-item {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 6px 10px;
  border-radius: 6px;
  cursor: pointer;
  transition: background 0.2s;
  font-size: 0.85rem;
}

.legend-item:hover,
.legend-item.active {
  background: var(--color-accent-light);
}

.legend-dot {
  width: 10px;
  height: 10px;
  border-radius: 50%;
  flex-shrink: 0;
}

.legend-name {
  color: var(--color-text-primary);
  font-weight: 500;
  flex: 1;
}

.legend-pct {
  color: var(--color-text-muted);
  font-size: 0.8rem;
  font-variant-numeric: tabular-nums;
}

.pie-detail {
  position: absolute;
  background: rgba(44, 62, 80, 0.94);
  color: #fff;
  padding: 14px 18px;
  border-radius: 10px;
  font-size: 0.82rem;
  pointer-events: none;
  z-index: 20;
  box-shadow: 0 8px 24px rgba(0,0,0,0.18);
  min-width: 200px;
  max-width: 280px;
  backdrop-filter: blur(8px);
}

.detail-title {
  font-weight: 600;
  font-size: 0.95rem;
  margin-bottom: 4px;
}

.detail-total {
  font-size: 1.1rem;
  font-weight: 700;
  color: #8ecae6;
  margin-bottom: 10px;
  font-variant-numeric: tabular-nums;
}

.detail-items {
  border-top: 1px solid rgba(255,255,255,0.15);
  padding-top: 8px;
}

.detail-item {
  display: flex;
  justify-content: space-between;
  gap: 24px;
  padding: 3px 0;
  white-space: nowrap;
}

.detail-item-name {
  opacity: 0.9;
}

.detail-item-pct {
  font-weight: 500;
  font-variant-numeric: tabular-nums;
}

.detail-empty {
  opacity: 0.6;
  font-size: 0.78rem;
}

@media (max-width: 768px) {
  .pie-layout {
    flex-direction: column;
    gap: 24px;
  }

  .pie-canvas-wrap {
    width: 280px;
    height: 280px;
  }

  .pie-legend {
    flex-direction: row;
    flex-wrap: wrap;
    justify-content: center;
    min-width: auto;
  }
}
</style>
