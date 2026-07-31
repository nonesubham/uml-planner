<template>
  <div class="uml-group" :style="{ width: data.w + 'px', height: data.h + 'px' }">
    <div class="uml-group-label">{{ data.label }}</div>
  </div>
</template>

<script setup>
defineProps({
  id: { type: [String, Number], default: '' },
  data: { type: Object, default: () => ({}) },
})
</script>

<style scoped>
.uml-group {
  position: relative;
  box-sizing: border-box;
  border: 1.5px dashed #4a4a4a;
  border-radius: 10px;
  background: rgba(255, 255, 255, 0.02);
  cursor: grab;
}
.uml-group-label {
  position: absolute;
  top: -11px;
  left: 12px;
  padding: 2px 12px;
  background: #1a1a1a;
  border: 1px solid #4a4a4a;
  border-radius: 6px;
  color: #bbb;
  font-size: 12px;
  font-weight: 600;
  letter-spacing: 0.03em;
  white-space: nowrap;
  max-width: calc(100% - 24px);
  overflow: hidden;
  text-overflow: ellipsis;
  cursor: grab;
  user-select: none;
}
</style>
