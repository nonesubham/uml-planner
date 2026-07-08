<template>
  <div class="relative w-full h-screen bg-black">
    <div class="absolute top-4 left-4 z-20 toolbar">
      <button @click="showAddClassModal = true" class="btn-primary">+ Class</button>
      <button @click="startNew" class="btn-secondary">New</button>
      <button @click="saveFile" class="btn-secondary" :disabled="!nodes.length && !edges.length">Save</button>
      <button @click="openFileInput" class="btn-secondary">Open</button>
      <div class="relative" @click.stop>
        <button @click="showExportMenu = !showExportMenu" class="btn-secondary">Export ▾</button>
        <div v-if="showExportMenu" class="dropdown-menu">
          <button @click="exportPng(); showExportMenu = false" class="dropdown-item">PNG</button>
          <button @click="exportJpg(); showExportMenu = false" class="dropdown-item">JPG</button>
          <button @click="exportSvg(); showExportMenu = false" class="dropdown-item">SVG</button>
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
      @move="onMove"
      @node-context-menu="onNodeContextMenu"
      @pane-context-menu="onPaneContextMenu"
      @edge-context-menu="onEdgeContextMenu"
      :default-edge-options="defaultEdgeOptions"
      class="vue-flow-custom"
    >
      <template #node-uml-class="nodeProps">
        <UMLClassNode v-bind="nodeProps" @delete="deleteNode(nodeProps.id)" />
      </template>
      <template #edge-uml-edge="edgeProps">
        <UMLClassEdge v-bind="edgeProps" />
      </template>


      <Background :gap="20" :pattern-color="'#333'" :size="2" />
      <Controls show-zoom show-fit-view>
        <button @click="showArrowheads = !showArrowheads" class="ctrl-btn" :class="{ active: showArrowheads }" title="Toggle arrow heads">
          <svg width="16" height="16" viewBox="0 0 16 16" fill="none">
            <path d="M2 8h10" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/>
            <path d="M10 5l3 3-3 3" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
          </svg>
        </button>
      </Controls>
      <div class="zoom-level">{{ Math.round(viewport.zoom * 100) }}%</div>
    </VueFlow>

    <div v-if="showRelationModal" class="fixed inset-0 z-50 flex items-center justify-center bg-black/80">
      <div class="bg-neutral-900 rounded-lg shadow-2xl w-80 max-w-full mx-4 border border-gray-700">
        <div class="p-6">
          <h2 class="text-lg font-bold text-white mb-4">{{ editingEdgeId ? 'Change Relation Type' : 'Relation Type' }}</h2>
          <div class="mb-4">
            <label class="block text-sm font-medium text-gray-300 mb-2">Custom Label (optional):</label>
            <input v-model="customEdgeLabel" type="text" placeholder="e.g. communicates with"
              class="w-full px-3 py-2 bg-black text-white border border-gray-600 rounded focus:ring-2 focus:ring-white focus:border-white placeholder-gray-500 text-sm" />
          </div>
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
              <div v-for="(attr, i) in form.attributes" :key="i" class="flex items-center gap-1 mb-1.5">
                <select v-model="attr.access"
                  class="w-14 shrink-0 px-1 py-1 bg-black text-white border border-gray-600 rounded text-xs focus:ring-2 focus:ring-white focus:border-white">
                  <option value="public">+</option><option value="private">-</option><option value="protected">#</option>
                </select>
                <input v-model="attr.name" type="text" placeholder="name"
                  class="min-w-0 flex-1 w-0 px-1.5 py-1 bg-black text-white border border-gray-600 rounded text-xs focus:ring-2 focus:ring-white focus:border-white placeholder-gray-500">
                <span class="text-gray-400 shrink-0 text-xs">:</span>
                <input v-model="attr.type" type="text" placeholder="type"
                  class="min-w-0 flex-1 w-0 px-1.5 py-1 bg-black text-white border border-gray-600 rounded text-xs focus:ring-2 focus:ring-white focus:border-white placeholder-gray-500">
                <button type="button" @click="form.attributes.splice(i, 1)"
                  class="shrink-0 bg-white text-black h-6 w-6 flex items-center justify-center text-xs rounded hover:bg-gray-300">×</button>
              </div>
              <button type="button" @click="form.attributes.push({ name: '', type: '', access: 'private' })"
                class="btn-secondary mt-1">+ Add Attribute</button>
            </div>

            <div class="mb-6">
              <label class="block text-sm font-medium text-gray-300 mb-2">Methods (name(params): returnType):</label>
              <div v-for="(method, i) in form.methods" :key="i" class="flex items-center gap-1 mb-1.5">
                <select v-model="method.access"
                  class="w-14 shrink-0 px-1 py-1 bg-black text-white border border-gray-600 rounded text-xs focus:ring-2 focus:ring-white focus:border-white">
                  <option value="public">+</option><option value="private">-</option><option value="protected">#</option>
                </select>
                <input v-model="method.name" type="text" placeholder="name"
                  class="min-w-0 flex-1 w-0 px-1.5 py-1 bg-black text-white border border-gray-600 rounded text-xs focus:ring-2 focus:ring-white focus:border-white placeholder-gray-500">
                <span class="text-gray-400 shrink-0 text-xs">(</span>
                <input v-model="method.params" type="text" placeholder="params"
                  class="min-w-0 flex-1 w-0 px-1.5 py-1 bg-black text-white border border-gray-600 rounded text-xs focus:ring-2 focus:ring-white focus:border-white placeholder-gray-500">
                <span class="text-gray-400 shrink-0 text-xs">):</span>
                <input v-model="method.returnType" type="text" placeholder="type"
                  class="min-w-0 flex-1 w-0 px-1.5 py-1 bg-black text-white border border-gray-600 rounded text-xs focus:ring-2 focus:ring-white focus:border-white placeholder-gray-500">
                <button type="button" @click="form.methods.splice(i, 1)"
                  class="shrink-0 bg-white text-black h-6 w-6 flex items-center justify-center text-xs rounded hover:bg-gray-300">×</button>
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
    <div
      v-if="contextMenu"
      class="context-menu"
      :style="{ left: contextMenu.x + 'px', top: contextMenu.y + 'px' }"
      @click.stop
    >
      <template v-if="contextMenu.edge">
        <div class="context-label">Line Style</div>
        <button
          v-for="t in relationTypes" :key="t.value"
          @click="changeEdgeType(t.value)"
          class="context-item"
          :class="{ active: contextMenu.edge.label === t.label }"
        >{{ t.symbol }} {{ t.label }}</button>
        <div class="context-divider"></div>
        <button @click="editEdgeLabel()" class="context-item">Edit Label</button>
        <button @click="deleteEdge(contextMenu.edge.id)" class="context-item danger">Delete</button>
      </template>
      <template v-else>
        <button
          @click="copySelected()"
          class="context-item"
          :disabled="!contextMenu.node"
        >Copy</button>
        <button
          @click="pasteNodes()"
          class="context-item"
          :disabled="!copiedNodes"
        >Paste</button>
        <button
          @click="editNodeFromMenu()"
          class="context-item"
          :disabled="!contextMenu.node"
        >Edit</button>
        <button
          @click="deleteSelected()"
          class="context-item danger"
          :disabled="!contextMenu.node"
        >Delete</button>
      </template>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick, provide } from 'vue'
import { VueFlow, useVueFlow } from '@vue-flow/core'
import { Background } from '@vue-flow/background'
import { Controls } from '@vue-flow/controls'
import UMLClassNode from './UMLClassNode.vue'
import UMLClassEdge from './UMLClassEdge.vue'

const NODE_W = 200
const ROW = 22
const HEADER = 36

function accessSym(a) {
  if (a === 'public') return '+'
  if (a === 'private') return '-'
  if (a === 'protected') return '#'
  return ' '
}

function nodeH(data) {
  let h = HEADER
  if (data.attributes?.length) { h += 2; h += data.attributes.length * ROW }
  if (data.methods?.length) { h += 2; h += data.methods.length * ROW }
  return h
}

function handlePt(node, handleId) {
  const x = node.position.x, y = node.position.y, w = NODE_W, h = nodeH(node.data)
  if (handleId === 'left') return { x, y: y + h / 2 }
  if (handleId === 'right') return { x: x + w, y: y + h / 2 }
  if (handleId === 'top') return { x: x + w / 2, y }
  if (handleId === 'bottom') return { x: x + w / 2, y: y + h }
  return { x: x + w / 2, y: y + h / 2 }
}

function angle(a, b) {
  return Math.atan2(b.y - a.y, b.x - a.x)
}

function drawArrow(ctx, x, y, ang, size) {
  ctx.beginPath()
  ctx.moveTo(x, y)
  ctx.lineTo(x - size * Math.cos(ang - 0.4), y - size * Math.sin(ang - 0.4))
  ctx.lineTo(x - size * Math.cos(ang + 0.4), y - size * Math.sin(ang + 0.4))
  ctx.closePath()
  ctx.fill()
}

function drawHollowTriangle(ctx, x, y, ang, size) {
  ctx.beginPath()
  ctx.moveTo(x, y)
  ctx.lineTo(x - size * Math.cos(ang - 0.5), y - size * Math.sin(ang - 0.5))
  ctx.lineTo(x - size * Math.cos(ang), y - size * Math.sin(ang) - size * 0.5)
  ctx.lineTo(x - size * Math.cos(ang + 0.5), y - size * Math.sin(ang + 0.5))
  ctx.closePath()
  ctx.stroke()
}

function drawDiamond(ctx, x, y, ang, size, fill) {
  ctx.beginPath()
  ctx.moveTo(x, y)
  ctx.lineTo(x - size * Math.cos(ang - 0.6), y - size * Math.sin(ang - 0.6))
  ctx.lineTo(x - 2 * size * Math.cos(ang), y - 2 * size * Math.sin(ang))
  ctx.lineTo(x - size * Math.cos(ang + 0.6), y - size * Math.sin(ang + 0.6))
  ctx.closePath()
  if (fill) ctx.fill()
  ctx.stroke()
}

function edgeTypeVal(label) {
  const entry = relationTypes.find(r => r.label === label)
  return entry ? entry.value : 'association'
}

function drawEdge(ctx, edge, nodesMap) {
  const src = nodesMap[edge.source]
  const dst = nodesMap[edge.target]
  if (!src || !dst) return

  const from = handlePt(src, edge.sourceHandle || 'right')
  const to = handlePt(dst, edge.targetHandle || 'left')
  const label = edge.label || 'has a'
  const typeVal = edgeTypeVal(label)

  const mx = (from.x + to.x) / 2, my = (from.y + to.y) / 2
  const ang = angle(from, to)
  const isDashed = typeVal === 'dependency'

  ctx.strokeStyle = '#555'
  ctx.fillStyle = '#555'
  ctx.lineWidth = 2
  ctx.setLineDash(isDashed ? [8, 5] : [])
  ctx.beginPath()
  ctx.moveTo(from.x, from.y)
  ctx.lineTo(to.x, to.y)
  ctx.stroke()
  ctx.setLineDash([])

  if (showArrowheads.value) {
    if (typeVal === 'inheritance') drawHollowTriangle(ctx, to.x, to.y, ang, 10)
    else if (typeVal === 'composition') drawDiamond(ctx, from.x, from.y, ang, 5, true)
    else if (typeVal === 'aggregation') drawDiamond(ctx, from.x, from.y, ang, 5, false)
    else if (typeVal === 'dependency' || typeVal === 'association') drawArrow(ctx, to.x, to.y, ang, 8)
  }

  ctx.fillStyle = '#fff'
  ctx.font = '11px sans-serif'
  ctx.textBaseline = 'bottom'
  ctx.textAlign = 'center'
  ctx.fillText(label, mx, my - 6)
}

function drawNode(ctx, node) {
  const d = node.data, x = node.position.x, y = node.position.y, w = NODE_W, h = nodeH(d)

  ctx.fillStyle = 'rgba(0,0,0,0.4)'
  ctx.beginPath()
  ctx.roundRect(x + 3, y + 3, w, h, 8)
  ctx.fill()

  ctx.fillStyle = '#1a1a1a'
  ctx.beginPath()
  ctx.roundRect(x, y, w, h, 8)
  ctx.fill()
  ctx.strokeStyle = '#333'
  ctx.lineWidth = 1
  ctx.beginPath()
  ctx.roundRect(x, y, w, h, 8)
  ctx.stroke()

  ctx.fillStyle = '#262626'
  ctx.beginPath()
  ctx.roundRect(x + 1, y + 1, w - 2, HEADER - 1, { upperLeft: 7, upperRight: 7 })
  ctx.fill()
  ctx.fillStyle = '#fff'
  ctx.font = 'bold 13px sans-serif'
  ctx.textBaseline = 'middle'
  ctx.textAlign = 'left'
  ctx.fillText(d.name, x + 12, y + HEADER / 2)

  let cy = y + HEADER
  const writeRow = (items, prefixFn) => {
    if (!items?.length) return
    cy += 1
    ctx.fillStyle = '#444'
    ctx.fillRect(x + 8, cy, w - 16, 1)
    cy += 1
    for (const item of items) {
      ctx.fillStyle = '#999'
      ctx.font = '11px monospace'
      ctx.textBaseline = 'middle'
      const txt = prefixFn
        ? `[${prefixFn(item)}] ${item.name}${item.type ? ': ' + item.type : ''}${item.params !== undefined ? '(' + item.params + '): ' + item.returnType : ''}`
        : item.name
      ctx.fillText(txt, x + 12, cy + ROW / 2)
      cy += ROW
    }
  }

  writeRow(d.attributes, (a) => accessSym(a.access))
  writeRow(d.methods, (m) => accessSym(m.access))
}

function getBounds(nodes) {
  if (!nodes.length) return { x: 0, y: 0, w: 400, h: 300 }
  let minX = Infinity, minY = Infinity, maxX = -Infinity, maxY = -Infinity
  for (const n of nodes) {
    const h = nodeH(n.data)
    minX = Math.min(minX, n.position.x)
    minY = Math.min(minY, n.position.y)
    maxX = Math.max(maxX, n.position.x + NODE_W)
    maxY = Math.max(maxY, n.position.y + h)
  }
  return { x: minX, y: minY, w: maxX - minX, h: maxY - minY }
}

function renderToCanvas(nodes, edges, bg) {
  const PAD = 40
  const bounds = getBounds(nodes)
  const W = bounds.w + PAD * 2
  const H = bounds.h + PAD * 2
  const scale = 3

  const canvas = document.createElement('canvas')
  canvas.width = W * scale
  canvas.height = H * scale
  const ctx = canvas.getContext('2d')
  ctx.scale(scale, scale)

  if (bg) {
    ctx.fillStyle = bg
    ctx.fillRect(0, 0, W, H)
  }

  ctx.translate(PAD - bounds.x, PAD - bounds.y)

  const nodesMap = Object.fromEntries(nodes.map(n => [n.id, n]))

  for (const edge of edges) drawEdge(ctx, edge, nodesMap)
  for (const node of nodes) drawNode(ctx, node)

  return canvas
}

async function exportPng() {
  const canvas = renderToCanvas(nodes.value, edges.value, null)
  const a = document.createElement('a')
  a.href = canvas.toDataURL('image/png')
  a.download = 'diagram.png'
  a.click()
}

async function exportJpg() {
  const canvas = renderToCanvas(nodes.value, edges.value, '#000')
  const a = document.createElement('a')
  a.href = canvas.toDataURL('image/jpeg', 0.95)
  a.download = 'diagram.jpg'
  a.click()
}

async function exportSvg() {
  const canvas = renderToCanvas(nodes.value, edges.value, null)
  const a = document.createElement('a')
  a.href = canvas.toDataURL('image/png')
  a.download = 'diagram.png'
  a.click()
}

const nodes = ref([])
const edges = ref([])
const showAddClassModal = ref(false)
const showRelationModal = ref(false)
const showExportMenu = ref(false)
const editingNodeId = ref(null)
const editingEdgeId = ref(null)
const pendingConnection = ref(null)
const fileInputRef = ref(null)
const contextMenu = ref(null)
const copiedNodes = ref(null)
const showArrowheads = ref(true)
provide('showArrowheads', showArrowheads)
provide('edges', edges)

const defaultEdgeOptions = { type: 'uml-edge' }

const customEdgeLabel = ref('')

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
      label: e.label, animated: e.animated, style: e.style
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
    edges.value = (data.edges || []).map(e => {
      const typeEntry = relationTypes.find(r => r.label === e.label)
      return { ...e, ...edgeStyleForType(typeEntry ? typeEntry.value : 'association') }
    })
    idCounter = Math.max(...data.nodes.map(n => parseInt(n.id.replace('node_', '')) || 0))
    savedViewport = data.viewport || null
  } catch {}
}

load()

const { screenToFlowCoordinate, viewport, setViewport, addSelectedNodes, getSelectedNodes } = useVueFlow()



onMounted(async () => {
  if (savedViewport) {
    await nextTick()
    setViewport(savedViewport)
  }
  document.addEventListener('click', () => {
    showExportMenu.value = false
    contextMenu.value = null
  })
  document.addEventListener('keydown', (e) => {
    if ((e.ctrlKey || e.metaKey) && e.key.toLowerCase() === 'a') {
      e.preventDefault()
      addSelectedNodes(nodes.value)
      contextMenu.value = null
    }
    if (e.key === 'Escape') {
      contextMenu.value = null
    }
    if ((e.key === 'Delete' || e.key === 'Del') && !showAddClassModal.value && !showRelationModal.value) {
      const selected = getSelectedNodes.value
      if (selected.length) {
        e.preventDefault()
        const ids = selected.map(n => n.id)
        edges.value = edges.value.filter(ed => !ids.includes(ed.source) && !ids.includes(ed.target))
        nodes.value = nodes.value.filter(n => !ids.includes(n.id))
        contextMenu.value = null
        save()
      }
    }
  }, { capture: true })
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

function onMove() {
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

function edgeMarker(typeVal) {
  const map = {
    association: { markerEnd: 'arrowclosed' },
    inheritance: { markerEnd: 'inheritance' },
    composition: { markerStart: 'composition' },
    aggregation: { markerStart: 'aggregation' },
    dependency: { markerEnd: 'arrow' },
  }
  return map[typeVal] || {}
}

function edgeStyleForType(typeVal) {
  const isDashed = typeVal === 'dependency'
  return {
    animated: isDashed,
    style: { stroke: '#666', ...(isDashed ? { strokeDasharray: '8 5' } : {}) },
    ...edgeMarker(typeVal),
  }
}

function confirmRelation(type) {
  const typeDef = relationTypes.find(r => r.value === type)
  const label = customEdgeLabel.value.trim() || (typeDef ? typeDef.label : type)
  const edgeProps = {
    label,
    labelBgPadding: [6, 3],
    labelBgBorderRadius: 4,
    ...edgeStyleForType(type)
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
  customEdgeLabel.value = ''
  save()
}

function cancelRelation() {
  pendingConnection.value = null
  editingEdgeId.value = null
  showRelationModal.value = false
  customEdgeLabel.value = ''
}

function onNodeDoubleClick(payload) {
  openModal(payload.node)
}

function onEdgeDoubleClick(payload) {
  editingEdgeId.value = payload.edge.id
  customEdgeLabel.value = ''
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
      animated: e.animated,
      style: e.style
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
      edges.value = (data.edges || []).map(e => {
        const typeEntry = relationTypes.find(r => r.label === e.label)
        return { ...e, ...edgeStyleForType(typeEntry ? typeEntry.value : 'association') }
      })
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

function onNodeContextMenu({ node, event }) {
  event.preventDefault()
  contextMenu.value = { x: event.clientX, y: event.clientY, node }
}

function onPaneContextMenu(event) {
  event.preventDefault()
  contextMenu.value = { x: event.clientX, y: event.clientY, node: null }
}

function onEdgeContextMenu({ edge, event }) {
  event.preventDefault()
  contextMenu.value = { x: event.clientX, y: event.clientY, edge }
}

function changeEdgeType(type) {
  const typeDef = relationTypes.find(r => r.value === type)
  if (!typeDef || !contextMenu.value?.edge) return
  const edgeId = contextMenu.value.edge.id
  edges.value = edges.value.map(e => e.id === edgeId ? { ...e, label: typeDef.label, ...edgeStyleForType(type) } : e)
  contextMenu.value = null
  save()
}

function editEdgeLabel() {
  if (!contextMenu.value?.edge) return
  editingEdgeId.value = contextMenu.value.edge.id
  customEdgeLabel.value = ''
  showRelationModal.value = true
  contextMenu.value = null
}

function deleteEdge(id) {
  edges.value = edges.value.filter(e => e.id !== id)
  contextMenu.value = null
  save()
}

function copySelected() {
  const selected = getSelectedNodes.value
  if (selected.length === 0 && contextMenu.value?.node) {
    copiedNodes.value = [JSON.parse(JSON.stringify(contextMenu.value.node))]
  } else if (selected.length > 0) {
    copiedNodes.value = selected.map(n => JSON.parse(JSON.stringify(n)))
  }
  contextMenu.value = null
}

function deleteSelected() {
  let ids
  const selected = getSelectedNodes.value
  if (contextMenu.value?.node && !selected.some(n => n.id === contextMenu.value.node.id)) {
    ids = [contextMenu.value.node.id]
  } else {
    ids = selected.map(n => n.id)
  }
  if (ids.length === 0 && contextMenu.value?.node) {
    ids = [contextMenu.value.node.id]
  }
  edges.value = edges.value.filter(e => !ids.includes(e.source) && !ids.includes(e.target))
  nodes.value = nodes.value.filter(n => !ids.includes(n.id))
  contextMenu.value = null
  save()
}

function pasteNodes() {
  if (!copiedNodes.value?.length) return
  const center = screenToFlowCoordinate({ x: window.innerWidth / 2, y: window.innerHeight / 2 })
  const pasted = copiedNodes.value.map((src, i) => {
    const offset = (i + 1) * 30
    return {
      ...JSON.parse(JSON.stringify(src)),
      id: nextId(),
      position: {
        x: (contextMenu.value?.x != null ? screenToFlowCoordinate({ x: contextMenu.value.x, y: contextMenu.value.y }).x : center.x) + offset - 80,
        y: (contextMenu.value?.y != null ? screenToFlowCoordinate({ x: contextMenu.value.x, y: contextMenu.value.y }).y : center.y) + offset - 40,
      },
      selected: true,
    }
  })
  nodes.value.push(...pasted)
  addSelectedNodes(pasted)
  contextMenu.value = null
  save()
}

function editNodeFromMenu() {
  if (!contextMenu.value?.node) return
  openModal(contextMenu.value.node)
  contextMenu.value = null
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
  padding: 7px 18px;
  background: #fff;
  color: #0a0a0a;
  border-radius: 8px;
  font-weight: 600;
  font-size: 13px;
  letter-spacing: 0.01em;
  border: none;
  cursor: pointer;
  transition: all 0.15s;
}
.btn-primary:hover { background: #e0e0e0; }

.btn-secondary {
  padding: 7px 18px;
  background: #1e1e1e;
  color: #aaa;
  border-radius: 8px;
  font-size: 13px;
  font-weight: 500;
  border: 1px solid #333;
  cursor: pointer;
  transition: all 0.15s;
}
.btn-secondary:hover:not(:disabled) {
  background: #2a2a2a;
  color: #eee;
  border-color: #555;
}
.btn-secondary:disabled {
  opacity: 0.4;
  cursor: default;
}
.btn-secondary.active {
  background: #333;
  color: #fff;
  border-color: #666;
}

.toolbar { display: flex; gap: 6px; }

.dropdown-menu {
  position: absolute;
  top: calc(100% + 6px);
  left: 0;
  z-index: 30;
  display: flex;
  flex-direction: column;
  background: #1a1a1a;
  border: 1px solid #333;
  border-radius: 8px;
  overflow: hidden;
  min-width: 120px;
  box-shadow: 0 8px 24px rgba(0,0,0,0.6);
}
.dropdown-item {
  padding: 8px 18px;
  text-align: left;
  color: #aaa;
  font-size: 13px;
  transition: all 0.12s;
  border: none;
  background: transparent;
  cursor: pointer;
}
.dropdown-item:hover { background: #2a2a2a; color: #eee; }
.dropdown-item + .dropdown-item { border-top: 1px solid #222; }

.zoom-level {
  position: absolute;
  bottom: 12px;
  right: 12px;
  z-index: 10;
  background: #1a1a1a;
  color: #888;
  font-size: 11px;
  font-family: monospace;
  padding: 2px 8px;
  border-radius: 4px;
  border: 1px solid #333;
  pointer-events: none;
  user-select: none;
}

.vue-flow__controls .ctrl-btn { color: #666; }
.vue-flow__controls .ctrl-btn:hover { color: #fff; }
.vue-flow__controls .ctrl-btn.active {
  background: #444;
  color: #fff;
}


.context-menu {
  position: fixed;
  z-index: 100;
  background: #1a1a1a;
  border: 1px solid #333;
  border-radius: 8px;
  overflow: hidden;
  min-width: 140px;
  box-shadow: 0 8px 32px rgba(0,0,0,0.7);
  padding: 4px;
}
.context-item {
  display: block;
  width: 100%;
  text-align: left;
  padding: 7px 14px;
  background: none;
  border: none;
  color: #ccc;
  font-size: 13px;
  cursor: pointer;
  border-radius: 4px;
  transition: background 0.12s;
}
.context-item:hover:not(:disabled) {
  background: #2a2a2a;
  color: #fff;
}
.context-item:disabled {
  color: #555;
  cursor: default;
  pointer-events: none;
}
.context-item.danger:hover {
  background: #3a1a1a;
  color: #f66;
}
.context-item.active {
  color: #fff;
  background: #333;
}
.context-label {
  padding: 4px 14px 2px;
  font-size: 11px;
  color: #666;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}
.context-divider {
  height: 1px;
  background: #333;
  margin: 4px 0;
}
</style>
