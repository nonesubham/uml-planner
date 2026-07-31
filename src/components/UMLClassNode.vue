<template>
  <div class="uml-node" :class="animClass" @animationend="animClass = ''" :style="{ background: '#1a1a1a', border: '1px solid #333', borderRadius: '8px', minWidth: '160px' }">
    <div :style="{ borderRadius: '8px', overflow: 'hidden' }">
      <div class="node-header px-3 py-2 flex items-center" :style="{ background: '#262626', borderBottom: '1px solid #333' }">
        <span class="font-bold text-white text-sm">{{ data.name }}</span>
        <span class="ml-auto text-xs text-gray-400 mr-2">{{ accessSymbol(data.access) }}</span>
        <button @click.stop="$emit('delete')" class="text-gray-500 hover:text-red-400 text-xs leading-none" title="Delete node">&times;</button>
      </div>

      <div v-if="data.attributes?.length" class="border-t" :style="{ borderColor: '#333' }">
        <div v-for="(attr, i) in data.attributes" :key="i" class="px-3 py-0.5 border-b last:border-b-0" :style="{ borderColor: '#333' }">
          <span class="font-mono text-gray-300 text-xs">[{{ accessSymbol(attr.access) }}] {{ attr.name }}: {{ attr.type }}</span>
        </div>
      </div>

      <div v-if="data.methods?.length" class="border-t" :style="{ borderColor: '#333' }">
        <div v-for="(method, i) in data.methods" :key="i" class="px-3 py-0.5 border-b last:border-b-0" :style="{ borderColor: '#333' }">
          <span class="font-mono text-gray-300 text-xs">[{{ accessSymbol(method.access) }}] {{ method.name }}({{ method.params }}): {{ method.returnType }}</span>
        </div>
      </div>
    </div>

    <Handle type="source" :position="Position.Left" id="left" />
    <Handle type="source" :position="Position.Right" id="right" />
    <Handle type="source" :position="Position.Top" id="top" />
    <Handle type="source" :position="Position.Bottom" id="bottom" />
  </div>
</template>

<script setup>
import { ref, inject, watch } from 'vue'
import { Handle, Position } from '@vue-flow/core'

const props = defineProps({
  data: { type: Object, default: () => ({}) },
  id: { type: [String, Number], default: '' }
})
const emit = defineEmits(['delete'])

const snapState = inject('snapState', null)
const animClass = ref('')

watch(() => snapState?.[props.id], (v) => {
  if (v) animClass.value = v.dir === 'in' ? 'snap-in' : 'snap-out'
})

const accessSymbol = (access) => {
  switch (access) {
    case 'public': return '+'
    case 'private': return '-'
    case 'protected': return '#'
    default: return ' '
  }
}
</script>

<style scoped>
.snap-in {
  animation: uml-snap-in 260ms cubic-bezier(0.2, 0.8, 0.3, 1);
}
.snap-out {
  animation: uml-snap-out 240ms cubic-bezier(0.2, 0.8, 0.3, 1);
}
@keyframes uml-snap-in {
  0% { transform: scale(0.82); filter: brightness(1.8); }
  60% { transform: scale(1.07); filter: brightness(1.2); }
  100% { transform: scale(1); filter: none; }
}
@keyframes uml-snap-out {
  0% { transform: scale(1.08); filter: brightness(1.8); }
  100% { transform: scale(1); filter: none; }
}
</style>
