<template>
  <div class="relative w-full h-screen bg-black">
    <div class="absolute top-4 left-4 z-20 flex gap-2">
      <button @click="showAddClassModal = true" class="btn-primary">+ Class</button>
      <button @click="startNew" class="btn-secondary">New</button>
      <button @click="saveFile" class="btn-secondary">Save</button>
      <button @click="openFileInput" class="btn-secondary">Open</button>
      <div class="relative" @click.stop>
        <button @click="showExportMenu = !showExportMenu" class="btn-secondary">Export ▾</button>
        <div v-if="showExportMenu" class="absolute top-full left-0 mt-1 z-30 flex flex-col bg-[#262626] border border-[#555] rounded-lg overflow-hidden shadow-lg min-w-[140px]">
          <button @click="exportPng(); showExportMenu = false" class="px-4 py-2 text-left text-[#ccc] hover:bg-[#333] transition text-sm border-b border-[#444] last:border-none">PNG</button>
          <button @click="exportJpg(); showExportMenu = false" class="px-4 py-2 text-left text-[#ccc] hover:bg-[#333] transition text-sm">JPG</button>
        </div>
      </div>
      <input ref="fileInputRef" type="file" accept=".umld" @change="loadFile" class="hidden" />
    </div>

    <VueFlow
      :nodes="nodes"
      :edges="edges"
      @connect="onConnect"
      @node-double-click="onNodeDoubleClick($event)"
      @edge-double-click="onEdgeDoubleClick($event)"
      @node-drag-stop="onNodeDragStop"
      :min-zoom="0.1"
      :max-zoom="4"
      :zoom-on-double-click="false"
      class="vue-flow-custom"
    >
      <template #node-uml-class="nodeProps">
        <UMLClassNode v-bind="nodeProps" @delete="deleteNode(nodeProps.id)" />
      </template>

      <Background :gap="20" :pattern-color="'#333'" :size="2" />
      <Controls show-zoom show-fit-view />
    </VueFlow>

    <div v-if="showRelationModal" class="fixed inset-0 z-50 flex items-center justify-center bg-black/80">
      <div class="bg-neutral-900 rounded-lg shadow-2xl w-80 max-w-full mx-4 border border-gray-700">
        <div class="p-6">
          <h2 class="text-lg font-bold text-white mb-4">{{ editingEdgeId ? 'Change Relation Type' : 'Relation Type' }}</h2>
          <div class="flex flex-col gap-2">
            <button v-for="type in relationTypes" :key="type.value" @click="confirmRelation(type.value)"
              class="flex items-center gap-3 px-4 py-3 bg-black text-white border border-gray-600 rounded hover:border-white transition-colors text-left">
              <span class="text-lg">{{ type.symbol }}</span>
              <div>
                <div class="font-medium text-sm">{{ type.label }}</div>
                <div class="text-xs text-gray-400">{{ type.desc }}</div>
              </div>
            </button>
          </div>
          <button @click="cancelRelation" class="mt-4 w-full px-4 py-2 bg-neutral-800 text-gray-400 rounded border border-gray-600 hover:bg-neutral-700 transition-colors text-sm">
            Cancel
          </button>
        </div>
      </div>
    </div>

    <div v-if="showAddClassModal" class="fixed inset-0 z-50 flex items-center justify-center bg-black/80">
      <div class="bg-neutral-900 rounded-lg shadow-2xl w-[600px] max-w-full mx-4 border border-gray-700 max-h-[80vh] flex flex-col">
        <div class="p-6 overflow-y-auto">
          <h2 class="text-xl font-bold text-white mb-6">{{ editingNodeId ? 'Edit Class' : 'Add Class' }}</h2>

          <form @submit.prevent="addClass">
            <div class="mb-4">
              <label class="block text-sm font-medium text-gray-300 mb-2">Class Name:</label>
              <input v-model="form.name" type="text" required placeholder="e.g. User"
                class="w-full px-3 py-2 bg-black text-white border border-gray-600 rounded focus:ring-2 focus:ring-white focus:border-white placeholder-gray-500" />
            </div>

            <div class="mb-4">
              <label class="block text-sm font-medium text-gray-300 mb-2">Access Modifier:</label>
              <select v-model="form.access"
                class="w-full px-3 py-2 bg-black text-white border border-gray-600 rounded focus:ring-2 focus:ring-white focus:border-white">
                <option value="public">+ Public</option>
                <option value="private">- Private</option>
                <option value="protected"># Protected</option>
              </select>
            </div>

            <div class="mb-6">
              <label class="block text-sm font-medium text-gray-300 mb-2">Attributes (name: type):</label>
              <div v-for="(attr, i) in form.attributes" :key="i" class="flex items-center gap-2 mb-2">
                <select v-model="attr.access"
                  class="w-20 px-1 py-1 bg-black text-white border border-gray-600 rounded text-sm focus:ring-2 focus:ring-white focus:border-white">
                  <option value="public">+</option><option value="private">-</option><option value="protected">#</option>
                </select>
                <input v-model="attr.name" type="text" placeholder="name"
                  class="flex-1 px-2 py-1 bg-black text-white border border-gray-600 rounded focus:ring-2 focus:ring-white focus:border-white placeholder-gray-500">
                <span class="text-gray-400">:</span>
                <input v-model="attr.type" type="text" placeholder="type"
                  class="flex-1 px-2 py-1 bg-black text-white border border-gray-600 rounded focus:ring-2 focus:ring-white focus:border-white placeholder-gray-500">
                <button type="button" @click="form.attributes.splice(i, 1)"
                  class="px-2 py-1 bg-white text-black h-8 w-8 flex items-center justify-center text-sm rounded hover:bg-gray-300">×</button>
              </div>
              <button type="button" @click="form.attributes.push({ name: '', type: '', access: 'private' })"
                class="btn-secondary">+ Add Attribute</button>
            </div>

            <div class="mb-6">
              <label class="block text-sm font-medium text-gray-300 mb-2">Methods (name(params): returnType):</label>
              <div v-for="(method, i) in form.methods" :key="i" class="flex items-center gap-2 mb-2">
                <select v-model="method.access"
                  class="w-20 px-1 py-1 bg-black text-white border border-gray-600 rounded text-sm focus:ring-2 focus:ring-white focus:border-white">
                  <option value="public">+</option><option value="private">-</option><option value="protected">#</option>
                </select>
                <input v-model="method.name" type="text" placeholder="name"
                  class="flex-1 px-2 py-1 bg-black text-white border border-gray-600 rounded focus:ring-2 focus:ring-white focus:border-white placeholder-gray-500">
                <span class="text-gray-400">(</span>
                <input v-model="method.params" type="text" placeholder="params"
                  class="flex-1 px-2 py-1 bg-black text-white border border-gray-600 rounded focus:ring-2 focus:ring-white focus:border-white placeholder-gray-500">
                <span class="text-gray-400">):</span>
                <input v-model="method.returnType" type="text" placeholder="returnType"
                  class="flex-1 px-2 py-1 bg-black text-white border border-gray-600 rounded focus:ring-2 focus:ring-white focus:border-white placeholder-gray-500">
                <button type="button" @click="form.methods.splice(i, 1)"
                  class="px-2 py-1 bg-white text-black h-8 w-8 flex items-center justify-center text-sm rounded hover:bg-gray-300">×</button>
              </div>
              <button type="button" @click="form.methods.push({ name: '', params: '', returnType: '', access: 'public' })"
                class="btn-secondary">+ Add Method</button>
            </div>

            <div class="flex justify-end gap-3 pt-2 border-t border-gray-700">
              <button type="button" @click="closeModal" class="btn-secondary">Cancel</button>
              <button type="submit" class="btn-primary">{{ editingNodeId ? 'Save' : 'Add Class' }}</button>
            </div>
          </form>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { VueFlow, useVueFlow } from '@vue-flow/core'
import { Background } from '@vue-flow/background'
import { Controls } from '@vue-flow/controls'
import { toPng, toJpeg } from 'html-to-image'
import UMLClassNode from './UMLClassNode.vue'

const nodes = ref([])
const edges = ref([])
const showAddClassModal = ref(false)
const showRelationModal = ref(false)
const showExportMenu = ref(false)
const editingNodeId = ref(null)
const editingEdgeId = ref(null)
const pendingConnection = ref(null)
const fileInputRef = ref(null)

const relationTypes = [
  { value: 'association', symbol: '───', label: 'has a', desc: 'Association' },
  { value: 'inheritance', symbol: '──▷', label: 'is a', desc: 'Inheritance / Generalization' },
  { value: 'composition', symbol: '◆──', label: 'contains', desc: 'Composition' },
  { value: 'aggregation', symbol: '◇──', label: 'has a (aggregate)', desc: 'Aggregation' },
  { value: 'dependency', symbol: '- - >', label: 'uses', desc: 'Dependency' },
]

const form = ref(createEmptyForm())

function createEmptyForm() {
  return {
    name: '',
    access: 'public',
    attributes: [{ name: '', type: '', access: 'private' }],
    methods: [{ name: '', params: '', returnType: '', access: 'public' }]
  }
}

let idCounter = 0
const nextId = () => `node_${++idCounter}`

const STORAGE_KEY = 'uml-planner-data'
let savedViewport = null

function save() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify({
    nodes: nodes.value.map(n => ({
      id: n.id,
      type: n.type,
      position: { x: n.position.x, y: n.position.y },
      data: { ...n.data }
    })),
    edges: edges.value.map(e => ({
      id: e.id, source: e.source, target: e.target,
      sourceHandle: e.sourceHandle, targetHandle: e.targetHandle,
      label: e.label, animated: e.animated
    })),
    viewport: viewport.value
  }))
}

function load() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    if (!raw) return
    const data = JSON.parse(raw)
    if (!data.nodes?.length) return
    nodes.value = data.nodes
    edges.value = data.edges || []
    idCounter = Math.max(...data.nodes.map(n => parseInt(n.id.replace('node_', '')) || 0))
    savedViewport = data.viewport || null
  } catch {}
}

load()

const { screenToFlowCoordinate, viewport, setViewport } = useVueFlow()

onMounted(() => {
  if (savedViewport) {
    setTimeout(() => { setViewport(savedViewport) }, 200)
  }
  document.addEventListener('mousedown', (e) => {
    if (showExportMenu.value) showExportMenu.value = false
  }, { once: false })
})

function openModal(nodeData) {
  if (nodeData) {
    editingNodeId.value = nodeData.id
    form.value = {
      name: nodeData.data.name,
      access: nodeData.data.access,
      attributes: nodeData.data.attributes.map(a => ({ ...a })),
      methods: nodeData.data.methods.map(m => ({ ...m }))
    }
  } else {
    editingNodeId.value = null
    form.value = createEmptyForm()
  }
  showAddClassModal.value = true
}

function closeModal() {
  showAddClassModal.value = false
  editingNodeId.value = null
}

function deleteNode(id) {
  edges.value = edges.value.filter(e => e.source !== id && e.target !== id)
  nodes.value = nodes.value.filter(n => n.id !== id)
  save()
}

function onNodeDragStop(dragEvent) {
  const dragged = dragEvent.node
  const node = nodes.value.find(n => n.id === dragged.id)
  if (node) {
    node.position = { x: dragged.position.x, y: dragged.position.y }
  }
  save()
}

function addClass() {
  if (!form.value.name.trim()) return
  const attrs = form.value.attributes.filter(a => a.name.trim() && a.type.trim())
  const meths = form.value.methods.filter(m => m.name.trim())

  if (editingNodeId.value) {
    const node = nodes.value.find(n => n.id === editingNodeId.value)
    if (node) {
      node.data = { ...node.data, name: form.value.name, access: form.value.access, attributes: attrs, methods: meths }
    }
  } else {
    const center = screenToFlowCoordinate({ x: window.innerWidth / 2, y: window.innerHeight / 2 })
    const offsets = [
      { x: 0, y: 0 },
      { x: 150, y: 0 },
      { x: 0, y: 150 },
      { x: -150, y: 0 },
      { x: 0, y: -150 },
      { x: 150, y: 150 },
      { x: -150, y: 150 },
      { x: 150, y: -150 },
      { x: -150, y: -150 },
    ]
    let pos = null
    for (let i = 0; i < offsets.length; i++) {
      const test = { x: center.x + offsets[i].x - 80, y: center.y + offsets[i].y - 40 }
      const taken = nodes.value.some(n => Math.abs(n.position.x - test.x) < 180 && Math.abs(n.position.y - test.y) < 180)
      if (!taken) { pos = test; break }
    }
    if (!pos) pos = { x: center.x + offsets[nodes.value.length % offsets.length].x - 80, y: center.y + offsets[nodes.value.length % offsets.length].y - 40 }
    nodes.value.push({
      id: nextId(),
      type: 'uml-class',
      position: pos,
      data: { name: form.value.name, access: form.value.access, attributes: attrs, methods: meths }
    })
  }
  save()
  closeModal()
}

function onConnect(connection) {
  pendingConnection.value = connection
  showRelationModal.value = true
}

function confirmRelation(type) {
  const typeDef = relationTypes.find(r => r.value === type)
  const edgeProps = {
    label: typeDef ? typeDef.label : type,
    labelBgPadding: [6, 3],
    labelBgBorderRadius: 4,
    animated: true,
    style: { stroke: '#666' }
  }

  if (editingEdgeId.value) {
    edges.value = edges.value.map(e => e.id === editingEdgeId.value ? { ...e, ...edgeProps } : e)
    editingEdgeId.value = null
  } else if (pendingConnection.value) {
    const conn = pendingConnection.value
    edges.value.push({
      id: `e${conn.source}-${conn.target}`,
      source: conn.source,
      target: conn.target,
      sourceHandle: conn.sourceHandle,
      targetHandle: conn.targetHandle,
      ...edgeProps
    })
  }

  pendingConnection.value = null
  showRelationModal.value = false
  save()
}

function cancelRelation() {
  pendingConnection.value = null
  editingEdgeId.value = null
  showRelationModal.value = false
}

function onNodeDoubleClick(payload) {
  openModal(payload.node)
}

function onEdgeDoubleClick(payload) {
  editingEdgeId.value = payload.edge.id
  showRelationModal.value = true
}

function startNew() {
  if (nodes.value.length === 0 && edges.value.length === 0) return
  if (!confirm('Start a new diagram? Current work will be lost.')) return
  nodes.value = []
  edges.value = []
  idCounter = 0
  savedViewport = null
  localStorage.removeItem(STORAGE_KEY)
}

function getDiagramData() {
  return {
    version: 1,
    nodes: nodes.value.map(n => ({
      id: n.id,
      type: n.type,
      position: { x: n.position.x, y: n.position.y },
      data: { ...n.data }
    })),
    edges: edges.value.map(e => ({
      id: e.id,
      source: e.source,
      target: e.target,
      sourceHandle: e.sourceHandle,
      targetHandle: e.targetHandle,
      label: e.label,
      animated: e.animated
    }))
  }
}

function saveFile() {
  const data = getDiagramData()
  const blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = 'diagram.umld'
  a.click()
  URL.revokeObjectURL(url)
}

function openFileInput() {
  fileInputRef.value?.click()
}

function loadFile(event) {
  const file = event.target.files?.[0]
  if (!file) return
  const reader = new FileReader()
  reader.onload = (e) => {
    try {
      const data = JSON.parse(e.target.result)
      if (!data.nodes) throw new Error('Invalid file')
      nodes.value = data.nodes
      edges.value = data.edges || []
      if (data.nodes.length > 0) {
        idCounter = Math.max(...data.nodes.map(n => parseInt(n.id.replace('node_', '')) || 0))
      }
      save()
    } catch (err) {
      alert('Failed to load file: ' + err.message)
    }
  }
  reader.readAsText(file)
  event.target.value = ''
}

async function exportPng() {
  const el = document.querySelector('.vue-flow__transformationpane')
  if (!el) return
  try {
    const dataUrl = await toPng(el, {
      backgroundColor: '#000',
      filter: (node) => !node.closest?.('.vue-flow__panel') && !node.classList?.contains('vue-flow__panel')
    })
    const a = document.createElement('a')
    a.href = dataUrl
    a.download = 'diagram.png'
    a.click()
  } catch (err) {
    alert('Failed to export PNG')
  }
}

async function exportJpg() {
  const el = document.querySelector('.vue-flow__transformationpane')
  if (!el) return
  try {
    const dataUrl = await toJpeg(el, {
      backgroundColor: '#000',
      quality: 0.95,
      filter: (node) => !node.closest?.('.vue-flow__panel') && !node.classList?.contains('vue-flow__panel')
    })
    const a = document.createElement('a')
    a.href = dataUrl
    a.download = 'diagram.jpg'
    a.click()
  } catch (err) {
    alert('Failed to export JPG')
  }
}
</script>

<style>
/* === Layout (structural, required) === */
.vue-flow {
  position: relative;
  width: 100%;
  height: 100%;
  overflow: hidden;
  z-index: 0;
  direction: ltr;
}
.vue-flow__container {
  position: absolute;
  height: 100%;
  width: 100%;
  left: 0;
  top: 0;
}
.vue-flow__pane { z-index: 1; }
.vue-flow__transformationpane {
  transform-origin: 0 0;
  z-index: 2;
  pointer-events: none;
}

.vue-flow__selection { z-index: 6; }
.vue-flow__edge-labels {
  position: absolute;
  width: 100%;
  height: 100%;
  pointer-events: none;
  -webkit-user-select: none;
  -moz-user-select: none;
  user-select: none;
}
.vue-flow__nodesselection-rect:focus,
.vue-flow__nodesselection-rect:focus-visible { outline: none; }
.vue-flow .vue-flow__edges {
  pointer-events: none;
  overflow: visible;
}
.vue-flow__edge {
  pointer-events: visibleStroke;
  cursor: pointer;
}
.vue-flow__edge.animated path {
  stroke-dasharray: 5;
  animation: dashdraw 0.5s linear infinite;
}
.vue-flow__edge.animated path.vue-flow__edge-interaction {
  stroke-dasharray: none;
  animation: none;
}
.vue-flow__edge.inactive { pointer-events: none; }
.vue-flow__edge.selected,
.vue-flow__edge:focus,
.vue-flow__edge:focus-visible { outline: none; }
.vue-flow__edge-textwrapper { pointer-events: all; }
.vue-flow__edge-text {
  pointer-events: none;
  -webkit-user-select: none;
  -moz-user-select: none;
  user-select: none;
}
.vue-flow__connection { pointer-events: none; }
.vue-flow__connection .animated {
  stroke-dasharray: 5;
  animation: dashdraw 0.5s linear infinite;
}
.vue-flow__connectionline { z-index: 1001; }
.vue-flow__nodes {
  pointer-events: none;
  transform-origin: 0 0;
}
.vue-flow__node {
  position: absolute;
  -webkit-user-select: none;
  -moz-user-select: none;
  user-select: none;
  pointer-events: all;
  transform-origin: 0 0;
  box-sizing: border-box;
}
.vue-flow__nodesselection {
  z-index: 3;
  transform-origin: left top;
  pointer-events: none;
}
.vue-flow__nodesselection-rect {
  position: absolute;
  pointer-events: all;
  cursor: grab;
}
.vue-flow__handle {
  position: absolute;
  pointer-events: none;
  min-width: 5px;
  min-height: 5px;
}
.vue-flow__handle-bottom {
  left: 50%;
  bottom: 0;
  transform: translate(-50%, 50%);
}
.vue-flow__handle-top {
  left: 50%;
  top: 0;
  transform: translate(-50%, -50%);
}
.vue-flow__handle-left {
  top: 50%;
  left: 0;
  transform: translate(-50%, -50%);
}
.vue-flow__handle-right {
  top: 50%;
  right: 0;
  transform: translate(50%, -50%);
}
.vue-flow__edgeupdater {
  cursor: move;
  pointer-events: all;
}
.vue-flow__panel {
  position: absolute;
  z-index: 5;
  margin: 15px;
}
.vue-flow__panel.top { top: 0; }
.vue-flow__panel.bottom { bottom: 0; }
.vue-flow__panel.left { left: 0; }
.vue-flow__panel.right { right: 0; }
.vue-flow__panel.center {
  left: 50%;
  transform: translateX(-50%);
}
@keyframes dashdraw {
  from { stroke-dashoffset: 10; }
}

/* === Custom black theme === */
.vue-flow-custom { background: #000; }

.vue-flow__pane.draggable { cursor: grab; }
.vue-flow__pane.selection { cursor: pointer; }
.vue-flow__pane.dragging { cursor: grabbing; }

.vue-flow__node {
  cursor: grab;
}
.vue-flow__node.dragging { cursor: grabbing; }

.vue-flow__node.selected .uml-node {
  border-color: #666 !important;
  box-shadow: 0 0 0 1px #444;
}

.vue-flow__edge-path,
.vue-flow__connection-path {
  stroke: #555;
  stroke-width: 2;
  fill: none;
}

.vue-flow__edge.selected .vue-flow__edge-path {
  stroke: #888;
}

.vue-flow__connection-path {
  stroke-dasharray: 5 3;
}

.vue-flow__handle {
  width: 10px;
  height: 10px;
  background: #555;
  border: 2px solid #1a1a1a;
  border-radius: 50%;
  transition: background 0.15s, box-shadow 0.15s;
}
.vue-flow__handle.connectable {
  cursor: crosshair;
  pointer-events: all;
}
.vue-flow__handle:hover,
.vue-flow__handle.connecting {
  background: #fff;
  box-shadow: 0 0 0 3px rgba(255, 255, 255, 0.3);
}

.vue-flow__background pattern circle {
  fill: #333;
}

.vue-flow__controls {
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.4);
  border: 1px solid #333;
  border-radius: 8px;
  overflow: hidden;
}
.vue-flow__controls button {
  background: #1a1a1a;
  border: none;
  border-bottom: 1px solid #2a2a2a;
  fill: #666;
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.15s, fill 0.15s;
  cursor: pointer;
}
.vue-flow__controls button:last-child { border-bottom: none; }
.vue-flow__controls button:hover {
  background: #333;
  fill: #fff;
}
.vue-flow__controls button:active {
  background: #444;
  fill: #fff;
}
.vue-flow__controls button svg {
  width: 16px;
  height: 16px;
  display: block;
}

.vue-flow__selection {
  background: rgba(255, 255, 255, 0.04);
  border: 1px solid rgba(255, 255, 255, 0.15);
}

.vue-flow__edge-textbg {
  fill: #1a1a1a;
  stroke: #555;
  stroke-width: 1;
}
.vue-flow__edge-text {
  fill: #fff;
  font-size: 11px;
  font-weight: 500;
  pointer-events: none;
  user-select: none;
}

.vue-flow__nodesselection-rect,
.vue-flow__selection {
  background: rgba(255, 255, 255, 0.04);
  border: 1px solid rgba(255, 255, 255, 0.15);
}

.btn-primary {
  padding: 8px 16px;
  background: #fff;
  color: #000;
  border-radius: 6px;
  font-weight: 500;
  transition: background 0.15s;
  border: none;
  cursor: pointer;
}
.btn-primary:hover { background: #ccc; }

.btn-secondary {
  padding: 8px 16px;
  background: #262626;
  color: #ccc;
  border-radius: 6px;
  border: 1px solid #555;
  transition: background 0.15s;
  cursor: pointer;
}
.btn-secondary:hover { background: #333; }
</style>
