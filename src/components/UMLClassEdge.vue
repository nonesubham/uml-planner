<template>
  <g>
    <BaseEdge
      :id="id"
      :path="path"
      :style="style"
      :label="relationLabel"
      :label-x="labelX"
      :label-y="labelY"
      :label-style="labelStyle"
      :label-bg-style="labelBgStyle"
      :label-bg-padding="labelBgPadding"
      :label-bg-border-radius="labelBgBorderRadius"
    />
  </g>
</template>

<script setup>
import { computed } from 'vue'
import { BaseEdge, getBezierPath } from '@vue-flow/core'

const props = defineProps({
  id: String,
  source: String,
  target: String,
  sourceX: Number,
  sourceY: Number,
  targetX: Number,
  targetY: Number,
  sourcePosition: String,
  targetPosition: String,
  style: Object,
  label: String,
  labelStyle: Object,
  labelBgStyle: Object,
  labelBgPadding: [Number, Array],
  labelBgBorderRadius: Number,
  animated: Boolean,
  curvature: Number,
})

const relationSymbols = {
  'has a': ['───', '───'],
  'is a': ['──▷', '▷──'],
  'contains': ['◆──', '──◆'],
  'has a (aggregate)': ['◇──', '──◇'],
  'uses': ['─ ─>', '<─ ─'],
}

const edgeDir = computed(() => {
  const fromPos = props.sourcePosition || 'bottom'
  const toPos = props.targetPosition || 'top'
  const isRightward =
    (fromPos === 'left' && toPos !== 'left') ||
    (fromPos === 'bottom' && toPos === 'top') ||
    (fromPos === 'top' && toPos === 'bottom') ||
    (fromPos === 'right' && toPos !== 'right')
  return isRightward ? 0 : 1
})

const relationLabel = computed(() => {
  const syms = relationSymbols[props.label]
  const sym = syms ? syms[edgeDir.value] : (edgeDir.value === 0 ? '───' : '───')
  return `${sym} ${props.label}`
})

const sx = computed(() => props.sourceX ?? 0)
const sy = computed(() => props.sourceY ?? 0)
const tx = computed(() => props.targetX ?? 0)
const ty = computed(() => props.targetY ?? 0)

const pathData = computed(() => {
  const result = getBezierPath({
    sourceX: sx.value,
    sourceY: sy.value,
    sourcePosition: props.sourcePosition || 'bottom',
    targetX: tx.value,
    targetY: ty.value,
    targetPosition: props.targetPosition || 'top',
    curvature: props.curvature ?? 0.25,
  })
  return { path: result[0], labelX: result[1], labelY: result[2] }
})

const path = computed(() => pathData.value.path)
const labelX = computed(() => pathData.value.labelX)
const labelY = computed(() => pathData.value.labelY)
</script>
