<template>
  <div class="skill-radar-shell">
    <v-chart
      class="skill-radar"
      :option="chartOption"
      autoresize
      @mouseover="handleMouseOver"
      @mousemove="handleMouseMove"
      @mouseout="handleMouseOut"
      @globalout="clearHover"
    />

    <div
      v-if="activePoint"
      class="point-highlight"
      :style="highlightStyle"
      aria-hidden="true"
    />

    <div
      v-if="activePoint"
      class="point-tooltip"
      :style="tooltipStyle"
      role="status"
      aria-live="polite"
    >
      <span class="point-tooltip__label">{{ activePoint.name }}</span>
      <strong class="point-tooltip__value">{{ activePoint.value }}</strong>
    </div>
  </div>
</template>

<script setup>
import { computed, reactive } from 'vue'
import VChart from 'vue-echarts'
import { use } from 'echarts/core'
import { RadarChart } from 'echarts/charts'
import { TooltipComponent, LegendComponent } from 'echarts/components'
import { CanvasRenderer } from 'echarts/renderers'

use([RadarChart, TooltipComponent, LegendComponent, CanvasRenderer])

const isDark = computed(() => document.documentElement.classList.contains('dark'))

const skillData = {
  indicator: [
    { name: 'Java核心', max: 100 },
    { name: '后端框架', max: 100 },
    { name: '数据存储', max: 100 },
    { name: '消息队列', max: 100 },
    { name: '大数据', max: 100 },
    { name: '架构设计', max: 100 },
    { name: '工程化', max: 100 },
    { name: 'AI技术', max: 100 }
  ],
  values: [95, 90, 88, 85, 82, 90, 85, 75]
}

const hoverState = reactive({
  pointIndex: null,
  pointX: 0,
  pointY: 0,
  tooltipX: 0,
  tooltipY: 0
})

const activePoint = computed(() => {
  if (hoverState.pointIndex === null) {
    return null
  }

  return {
    name: skillData.indicator[hoverState.pointIndex].name,
    value: skillData.values[hoverState.pointIndex]
  }
})

const tooltipStyle = computed(() => ({
  left: `${hoverState.tooltipX + 14}px`,
  top: `${hoverState.tooltipY - 12}px`
}))

const highlightStyle = computed(() => ({
  left: `${hoverState.pointX}px`,
  top: `${hoverState.pointY}px`
}))

function getPointIndex(params) {
  const pointIndex = params?.event?.target?.__dimIdx
  return Number.isInteger(pointIndex) ? pointIndex : null
}

function updateHover(params) {
  const pointIndex = getPointIndex(params)

  if (pointIndex === null) {
    return
  }

  const event = params.event
  const target = event?.target

  hoverState.pointIndex = pointIndex
  hoverState.pointX = typeof target?.x === 'number' ? target.x : event.offsetX
  hoverState.pointY = typeof target?.y === 'number' ? target.y : event.offsetY
  hoverState.tooltipX = event.offsetX
  hoverState.tooltipY = event.offsetY
}

function clearHover() {
  hoverState.pointIndex = null
}

function handleMouseOver(params) {
  updateHover(params)
}

function handleMouseMove(params) {
  if (getPointIndex(params) === null) {
    return
  }

  updateHover(params)
}

function handleMouseOut(params) {
  if (getPointIndex(params) !== null) {
    clearHover()
  }
}

const chartOption = computed(() => {
  const dark = isDark.value
  const textColor = dark ? '#e2e8f0' : '#1e293b'
  const splitColor = dark ? 'rgba(148, 163, 184, 0.3)' : 'rgba(59, 130, 246, 0.3)'
  const areaColors = dark 
    ? ['rgba(96, 165, 250, 0.02)', 'rgba(96, 165, 250, 0.05)', 'rgba(96, 165, 250, 0.1)', 'rgba(96, 165, 250, 0.15)', 'rgba(96, 165, 250, 0.2)']
    : ['rgba(59, 130, 246, 0.02)', 'rgba(59, 130, 246, 0.05)', 'rgba(59, 130, 246, 0.1)', 'rgba(59, 130, 246, 0.15)', 'rgba(59, 130, 246, 0.2)']
  const mainColor = dark ? '#60a5fa' : '#3b82f6'
  
  return {
    backgroundColor: 'transparent',
    tooltip: {
      show: false
    },
    radar: {
      indicator: skillData.indicator,
      shape: 'polygon',
      center: ['50%', '55%'],
      radius: '65%',
      splitNumber: 5,
      axisName: {
        color: textColor,
        fontSize: 14,
        fontWeight: 'bold',
        padding: [3, 8],
        backgroundColor: dark ? 'rgba(30, 41, 59, 0.8)' : 'rgba(59, 130, 246, 0.1)',
        borderRadius: 4
      },
      splitLine: { lineStyle: { color: splitColor, width: 1 } },
      splitArea: { show: true, areaStyle: { color: areaColors } },
      axisLine: { lineStyle: { color: splitColor, width: 2 } }
    },
    series: [{
      type: 'radar',
      symbol: 'circle',
      symbolSize: 10,
      lineStyle: { width: 3, color: mainColor },
      areaStyle: { 
        color: {
          type: 'linear',
          x: 0, y: 0, x2: 1, y2: 1,
          colorStops: [
            { offset: 0, color: mainColor + 'cc' },
            { offset: 0.5, color: 'rgba(168, 85, 247, 0.6)' },
            { offset: 1, color: 'rgba(236, 72, 153, 0.8)' }
          ]
        }
      },
      itemStyle: { 
        color: mainColor, 
        borderColor: mainColor, 
        borderWidth: 3 
      },
      emphasis: {
        disabled: true
      },
      label: { show: false },
      data: [{
        value: skillData.values,
        name: '当前水平'
      }]
    }],
    animation: true,
    animationDuration: 2000,
    animationEasing: 'cubicOut'
  }
})
</script>

<style scoped>
.skill-radar-shell {
  position: relative;
  width: 100%;
  max-width: 800px;
  margin: 2rem auto;
}

.skill-radar {
  width: 100%;
  height: 480px;
}

.point-highlight {
  position: absolute;
  width: 24px;
  height: 24px;
  border-radius: 9999px;
  border: 3px solid rgba(255, 255, 255, 0.92);
  background: rgba(59, 130, 246, 0.95);
  box-shadow:
    0 0 0 6px rgba(59, 130, 246, 0.18),
    0 10px 24px rgba(59, 130, 246, 0.35);
  transform: translate(-50%, -50%);
  pointer-events: none;
}

.point-tooltip {
  position: absolute;
  display: flex;
  align-items: center;
  gap: 0.6rem;
  padding: 0.5rem 0.75rem;
  border: 1px solid rgba(59, 130, 246, 0.45);
  border-radius: 8px;
  background: rgba(15, 23, 42, 0.94);
  color: #f8fafc;
  box-shadow: 0 16px 40px rgba(15, 23, 42, 0.22);
  transform: translateY(-100%);
  pointer-events: none;
  white-space: nowrap;
  z-index: 2;
}

.point-tooltip__label {
  font-size: 0.85rem;
}

.point-tooltip__value {
  font-size: 0.95rem;
}

@media (max-width: 720px) {
  .skill-radar {
    height: 350px;
  }
}

@media (max-width: 480px) {
  .skill-radar {
    height: 280px;
  }

  .point-highlight {
    width: 20px;
    height: 20px;
  }

  .point-tooltip {
    gap: 0.45rem;
    padding: 0.45rem 0.65rem;
  }
}
</style>
