<script setup lang="ts">
import { onMounted, onBeforeUnmount, reactive, ref, computed } from 'vue'

const props = defineProps<{
  scale?: number
}>()

const scale = computed(() => props.scale ?? 1)
const stage = ref<HTMLElement | null>(null)

const BASE_WIDTH = 1100
const BASE_HEIGHT = 560

const clients = [
  { id: 'c1', label: 'Client A' }, 
  { id: 'c2', label: 'Client B' }, 
  { id: 'c3', label: 'Client C' }
]
const servers = [
  { id: 's1', label: 'Triton Server 1', models: ['GNN4ITk', 'Traccc'], gpus: 2, maxLoad: 2 },
  { id: 's2', label: 'Triton Server 2', models: ['ExaTrkX', 'Traccc'], gpus: 2, maxLoad: 2 }
]

// Refs
const clientRefs: Record<string, HTMLElement | null> = reactive({})
const entryRef = ref<HTMLElement | null>(null)
const lbRef = ref<HTMLElement | null>(null)
const serverRefs: Record<string, HTMLElement | null> = reactive({})
const monitorRef = ref<HTMLElement | null>(null)

// 路径数据
const paths = reactive({
  clientToEntry: {} as Record<string, string>,
  entryToLb: '',
  lbToServer: {} as Record<string, string>,
  serverToLb: {} as Record<string, string>,
  lbToEntry: '',
  entryToClient: {} as Record<string, string>,
  monitorLines: [] as string[]
})

// ===== 核心状态 =====
const serverLoad = reactive<Record<string, number>>({ s1: 0, s2: 0 })
const lbStats = reactive({ 
  total: 0, 
  s1: 0, 
  s2: 0,
  currentRouting: '',
  lastDecision: ''
})

// 活跃的请求动画
interface ActiveRequest {
  id: string
  clientId: string
  serverId: string
  color: string
  phase: 'ce' | 'el' | 'ls' | 'proc' | 'sl' | 'le' | 'ec' | 'done'
  progress: number // 0-1 当前阶段进度
  x: number
  y: number
}
const activeRequests = reactive<ActiveRequest[]>([])
let requestIdCounter = 0

// 请求颜色池
const colors = ['#fbbf24', '#38bdf8', '#a855f7', '#f472b6', '#22d3ee', '#fb923c', '#a3e635', '#f87171']
let colorIndex = 0

// ===== 路径计算 =====
function getRelativeRect(el: HTMLElement, root: HTMLElement) {
  let left = 0, top = 0
  let current: HTMLElement | null = el
  while (current && current !== root) {
    left += current.offsetLeft
    top += current.offsetTop
    current = current.offsetParent as HTMLElement | null
  }
  return { left, top, width: el.offsetWidth, height: el.offsetHeight, right: left + el.offsetWidth, bottom: top + el.offsetHeight }
}

function rightCenter(el: HTMLElement, root: HTMLElement) {
  const r = getRelativeRect(el, root)
  return { x: r.right, y: r.top + r.height / 2 }
}

function leftCenter(el: HTMLElement, root: HTMLElement) {
  const r = getRelativeRect(el, root)
  return { x: r.left, y: r.top + r.height / 2 }
}

function topCenter(el: HTMLElement, root: HTMLElement) {
  const r = getRelativeRect(el, root)
  return { x: r.left + r.width / 2, y: r.top }
}

function bottomCenter(el: HTMLElement, root: HTMLElement) {
  const r = getRelativeRect(el, root)
  return { x: r.left + r.width / 2, y: r.bottom }
}

function line(from: { x: number; y: number }, to: { x: number; y: number }) {
  return `M ${from.x.toFixed(1)} ${from.y.toFixed(1)} L ${to.x.toFixed(1)} ${to.y.toFixed(1)}`
}

function quad(from: { x: number; y: number }, to: { x: number; y: number }, bend = 0.12, sign = -1) {
  const dx = to.x - from.x
  const dy = to.y - from.y
  const cx = from.x + dx * 0.5
  const cy = (from.y + to.y) * 0.5 + sign * Math.max(Math.abs(dx), Math.abs(dy) * 0.5) * bend
  return `M ${from.x.toFixed(1)} ${from.y.toFixed(1)} Q ${cx.toFixed(1)} ${cy.toFixed(1)} ${to.x.toFixed(1)} ${to.y.toFixed(1)}`
}

// 解析路径获取点位置
function getPointOnPath(pathD: string, t: number): { x: number; y: number } {
  const match = pathD.match(/M ([\d.]+) ([\d.]+) (?:Q ([\d.]+) ([\d.]+) ([\d.]+) ([\d.]+)|L ([\d.]+) ([\d.]+))/)
  if (!match) return { x: 0, y: 0 }
  
  const x1 = parseFloat(match[1])
  const y1 = parseFloat(match[2])
  
  if (match[3]) {
    // Quadratic curve
    const cx = parseFloat(match[3])
    const cy = parseFloat(match[4])
    const x2 = parseFloat(match[5])
    const y2 = parseFloat(match[6])
    const mt = 1 - t
    return {
      x: mt * mt * x1 + 2 * mt * t * cx + t * t * x2,
      y: mt * mt * y1 + 2 * mt * t * cy + t * t * y2
    }
  } else {
    // Line
    const x2 = parseFloat(match[7])
    const y2 = parseFloat(match[8])
    return {
      x: x1 + (x2 - x1) * t,
      y: y1 + (y2 - y1) * t
    }
  }
}

function updatePaths() {
  if (!stage.value || !entryRef.value || !lbRef.value) return
  const root = stage.value

  const entryLeft = leftCenter(entryRef.value, root)
  const entryRight = rightCenter(entryRef.value, root)
  const lbLeft = leftCenter(lbRef.value, root)
  const lbRight = rightCenter(lbRef.value, root)

  for (const c of clients) {
    const cEl = clientRefs[c.id]
    if (!cEl) continue
    const cRight = rightCenter(cEl, root)
    const bendSign = c.id === 'c2' ? 0 : (c.id === 'c1' ? -1 : 1)
    paths.clientToEntry[c.id] = quad(cRight, entryLeft, 0.12, bendSign)
    paths.entryToClient[c.id] = quad(entryLeft, cRight, 0.08, -bendSign)
  }

  paths.entryToLb = line(entryRight, lbLeft)
  paths.lbToEntry = line(lbLeft, entryRight)

  for (const s of servers) {
    const sEl = serverRefs[s.id]
    if (!sEl) continue
    const sLeft = leftCenter(sEl, root)
    const bendSign = s.id === 's1' ? -1 : 1
    paths.lbToServer[s.id] = quad(lbRight, sLeft, 0.18, bendSign)
    paths.serverToLb[s.id] = quad(sLeft, lbRight, 0.12, -bendSign)
  }

  if (monitorRef.value) {
    const monitorTop = topCenter(monitorRef.value, root)
    const targets = [entryRef.value, lbRef.value, ...Object.values(serverRefs).filter(Boolean)]
    paths.monitorLines = targets.map(el => {
      if (!el) return ''
      const targetBottom = bottomCenter(el as HTMLElement, root)
      return `M ${monitorTop.x.toFixed(1)} ${monitorTop.y.toFixed(1)} L ${targetBottom.x.toFixed(1)} ${targetBottom.y.toFixed(1)}`
    }).filter(Boolean)
  }
}

// ===== Load Balancer 逻辑 =====
function selectServer(): string {
  const load1 = serverLoad.s1
  const load2 = serverLoad.s2
  const max1 = servers[0].maxLoad
  const max2 = servers[1].maxLoad
  
  // 如果 s1 满载，选 s2
  if (load1 >= max1 && load2 < max2) {
    lbStats.lastDecision = `S1 full (${load1}/${max1}), route to S2`
    return 's2'
  }
  // 如果 s2 满载，选 s1
  if (load2 >= max2 && load1 < max1) {
    lbStats.lastDecision = `S2 full (${load2}/${max2}), route to S1`
    return 's1'
  }
  // 都满载，选负载较低的
  if (load1 >= max1 && load2 >= max2) {
    const target = load1 <= load2 ? 's1' : 's2'
    lbStats.lastDecision = `Both full! Queuing to ${target.toUpperCase()}`
    return target
  }
  // 都没满，选负载低的
  if (load1 <= load2) {
    lbStats.lastDecision = `S1 (${load1}) <= S2 (${load2}), route to S1`
    return 's1'
  } else {
    lbStats.lastDecision = `S2 (${load2}) < S1 (${load1}), route to S2`
    return 's2'
  }
}

// ===== 请求生命周期 =====
const phaseDurations: Record<string, number> = {
  ce: 500,   // client -> entry
  el: 350,   // entry -> lb
  ls: 450,   // lb -> server
  proc: 800, // processing
  sl: 450,   // server -> lb
  le: 350,   // lb -> entry
  ec: 500,   // entry -> client
}

function createRequest(clientId: string) {
  const serverId = selectServer()
  const color = colors[colorIndex % colors.length]
  colorIndex++
  
  const req: ActiveRequest = {
    id: `req_${++requestIdCounter}`,
    clientId,
    serverId,
    color,
    phase: 'ce',
    progress: 0,
    x: 0,
    y: 0
  }
  
  activeRequests.push(req)
  lbStats.total++
  lbStats[serverId as 's1' | 's2']++
  lbStats.currentRouting = `${clientId.toUpperCase()} → ${serverId.toUpperCase()}`
  
  return req
}

function getPathForPhase(req: ActiveRequest): string {
  switch (req.phase) {
    case 'ce': return paths.clientToEntry[req.clientId] || ''
    case 'el': return paths.entryToLb
    case 'ls': return paths.lbToServer[req.serverId] || ''
    case 'sl': return paths.serverToLb[req.serverId] || ''
    case 'le': return paths.lbToEntry
    case 'ec': return paths.entryToClient[req.clientId] || ''
    default: return ''
  }
}

function advancePhase(req: ActiveRequest) {
  const phases: ActiveRequest['phase'][] = ['ce', 'el', 'ls', 'proc', 'sl', 'le', 'ec', 'done']
  const currentIdx = phases.indexOf(req.phase)
  
  // 进入 processing 时增加负载
  if (req.phase === 'ls') {
    serverLoad[req.serverId]++
  }
  // 离开 processing 时减少负载
  if (req.phase === 'proc') {
    serverLoad[req.serverId] = Math.max(0, serverLoad[req.serverId] - 1)
  }
  
  req.phase = phases[currentIdx + 1] || 'done'
  req.progress = 0
}

// ===== 动画循环 =====
let animationFrame: number | null = null
let lastTime = 0

function animate(currentTime: number) {
  if (!lastTime) lastTime = currentTime
  const delta = currentTime - lastTime
  lastTime = currentTime

  for (const req of activeRequests) {
    if (req.phase === 'done') continue
    if (req.phase === 'proc') {
      // Processing 阶段只增加进度
      req.progress += delta / phaseDurations.proc
      if (req.progress >= 1) {
        advancePhase(req)
      }
      continue
    }

    const duration = phaseDurations[req.phase] || 500
    req.progress += delta / duration

    if (req.progress >= 1) {
      req.progress = 1
      advancePhase(req)
    }

    // 更新位置
    const pathD = getPathForPhase(req)
    if (pathD) {
      const pos = getPointOnPath(pathD, Math.min(req.progress, 1))
      req.x = pos.x
      req.y = pos.y
    }
  }

  // 清理已完成的请求
  const toRemove = activeRequests.filter(r => r.phase === 'done')
  for (const r of toRemove) {
    const idx = activeRequests.indexOf(r)
    if (idx >= 0) activeRequests.splice(idx, 1)
  }

  animationFrame = requestAnimationFrame(animate)
}

// ===== 请求生成器 =====
let requestTimers: number[] = []

function startRequestGenerator() {
  // Client A: 每 1.5s 发一个请求
  requestTimers.push(window.setInterval(() => createRequest('c1'), 1500))
  
  // Client B: 每 2s 发一个请求，延迟 500ms 开始
  setTimeout(() => {
    requestTimers.push(window.setInterval(() => createRequest('c2'), 2000))
  }, 500)
  
  // Client C: 每 1.8s 发一个请求，延迟 1000ms 开始
  setTimeout(() => {
    requestTimers.push(window.setInterval(() => createRequest('c3'), 1800))
  }, 1000)
}

// ===== 辅助函数 =====
function getLoadColor(load: number, max: number) {
  const ratio = load / max
  if (ratio === 0) return { text: 'text-emerald-400', bg: 'bg-emerald-500', border: 'border-sky-400/50', card: 'bg-sky-900/20', glow: '', status: 'Idle' }
  if (ratio < 0.5) return { text: 'text-emerald-400', bg: 'bg-emerald-500', border: 'border-emerald-400', card: 'bg-emerald-900/20', glow: 'shadow-emerald-500/20', status: 'Normal' }
  if (ratio < 1) return { text: 'text-yellow-400', bg: 'bg-yellow-500', border: 'border-yellow-400', card: 'bg-yellow-900/20', glow: 'shadow-yellow-500/20', status: 'Busy' }
  if (ratio === 1) return { text: 'text-orange-400', bg: 'bg-orange-500', border: 'border-orange-400', card: 'bg-orange-900/30', glow: 'shadow-orange-500/30', status: 'Full' }
  return { text: 'text-red-500', bg: 'bg-red-500', border: 'border-red-500', card: 'bg-red-900/30', glow: 'shadow-red-500/40', status: '⚠️ OVERLOAD' }
}

function getGpuPercent(load: number, max: number) {
  if (load === 0) return 10
  return Math.min(95, 30 + (load / max) * 65)
}

function isRequestPhase(phase: string) {
  return ['ce', 'el', 'ls'].includes(phase)
}

let ro: ResizeObserver | null = null

onMounted(() => {
  requestAnimationFrame(() => {
    updatePaths()
    animationFrame = requestAnimationFrame(animate)
    startRequestGenerator()
  })
  ro = new ResizeObserver(() => requestAnimationFrame(updatePaths))
  if (stage.value) ro.observe(stage.value)
  window.addEventListener('resize', updatePaths, { passive: true })
})

onBeforeUnmount(() => {
  if (ro && stage.value) ro.unobserve(stage.value)
  ro = null
  window.removeEventListener('resize', updatePaths)
  if (animationFrame) cancelAnimationFrame(animationFrame)
  requestTimers.forEach(t => clearInterval(t))
})

const avgLatency = computed(() => Math.round(12 + serverLoad.s1 * 8 + serverLoad.s2 * 6))
const totalLoad = computed(() => serverLoad.s1 + serverLoad.s2)
</script>

<template>
  <div 
    class="relative"
    :style="{ width: `${BASE_WIDTH * scale}px`, height: `${BASE_HEIGHT * scale}px` }"
  >
    <div class="origin-top-left" :style="{ transform: `scale(${scale})` }">
      <div 
        ref="stage" 
        class="relative bg-gradient-to-br from-[#0a192f] to-[#112240] p-6 rounded-xl shadow-2xl"
        :style="{ width: `${BASE_WIDTH}px`, height: `${BASE_HEIGHT}px` }"
      >
        <!-- ===== CLIENTS ===== -->
        <div class="absolute left-6 top-16 flex flex-col gap-5 w-36">
          <div 
            v-for="c in clients" 
            :key="c.id"
            :ref="el => (clientRefs[c.id] = el as HTMLElement)"
            class="relative border-2 border-cyan-400/60 rounded-xl px-4 py-3 text-center bg-cyan-900/20 backdrop-blur shadow-lg transition-all duration-300"
            :class="{ 'border-cyan-400 shadow-cyan-500/30': activeRequests.some(r => r.clientId === c.id) }"
          >
            <div class="text-cyan-100 font-bold text-sm">{{ c.label }}</div>
            <div class="text-cyan-400/60 text-xs mt-1">gRPC Client</div>
            <div 
              class="absolute -right-1 -top-1 w-2.5 h-2.5 rounded-full transition-colors duration-200"
              :class="activeRequests.some(r => r.clientId === c.id && r.phase !== 'done') ? 'bg-amber-400 animate-pulse' : 'bg-green-400'"
            ></div>
            <!-- 等待返回指示 -->
            <div 
              v-if="activeRequests.some(r => r.clientId === c.id && ['proc', 'sl', 'le', 'ec'].includes(r.phase))"
              class="absolute -bottom-1 left-1/2 -translate-x-1/2 text-xs text-emerald-400 animate-pulse"
            >
              ◀ waiting
            </div>
          </div>
          <div class="text-cyan-400 font-bold text-center tracking-widest text-xs mt-1">CLIENTS</div>
        </div>

        <!-- ===== ENTRY POINT ===== -->
        <div 
          ref="entryRef"
          class="absolute left-52 top-24 w-32 border-2 border-amber-400/70 rounded-2xl bg-amber-900/20 backdrop-blur p-4 shadow-lg transition-all duration-200"
          :class="{ 'shadow-amber-500/30 border-amber-400': activeRequests.some(r => ['ce', 'el', 'le', 'ec'].includes(r.phase)) }"
        >
          <div class="text-center">
            <div class="text-amber-300 text-2xl mb-2">🌐</div>
            <div class="text-amber-100 font-bold text-sm">Entry Point</div>
            <div class="text-amber-400/60 text-xs mt-1">API Gateway</div>
          </div>
          <div class="mt-3 flex justify-center gap-1">
            <div 
              v-for="i in 4" :key="i" 
              class="w-1.5 rounded-full transition-all duration-300"
              :class="activeRequests.length > 0 ? 'bg-amber-400/80 animate-pulse' : 'bg-amber-400/30'"
              :style="{ height: `${10 + i * 3}px`, animationDelay: `${i * 0.15}s` }"
            ></div>
          </div>
        </div>

        <!-- ===== LOAD BALANCER ===== -->
        <div 
          ref="lbRef"
          class="absolute left-96 top-14 w-48 border-2 border-purple-400/70 rounded-2xl bg-purple-900/20 backdrop-blur p-4 shadow-lg transition-all duration-200"
          :class="{ 'shadow-purple-500/40 border-purple-400': activeRequests.some(r => ['el', 'ls', 'sl', 'le'].includes(r.phase)) }"
        >
          <div class="text-center">
            <div class="text-purple-300 text-2xl mb-1">⚖️</div>
            <div class="text-purple-100 font-bold text-sm">Load Balancer</div>
            <div class="text-purple-400/60 text-xs">Least Connections</div>
          </div>
          
          <!-- 服务器状态 -->
          <div class="mt-3 grid grid-cols-2 gap-2">
            <div 
              v-for="s in servers" :key="s.id"
              class="text-center p-2 rounded border transition-all duration-300"
              :class="[
                getLoadColor(serverLoad[s.id], s.maxLoad).border.replace('border-', 'border-') + '/50',
                serverLoad[s.id] >= s.maxLoad ? 'bg-red-900/20' : 'bg-black/30'
              ]"
            >
              <div class="text-xs font-mono" :class="getLoadColor(serverLoad[s.id], s.maxLoad).text">
                {{ s.id.toUpperCase() }}
              </div>
              <div class="text-lg font-bold" :class="getLoadColor(serverLoad[s.id], s.maxLoad).text">
                {{ serverLoad[s.id] }}/{{ s.maxLoad }}
              </div>
              <div 
                class="text-xs mt-0.5"
                :class="serverLoad[s.id] >= s.maxLoad ? 'text-red-400 font-bold' : 'text-gray-500'"
              >
                {{ serverLoad[s.id] >= s.maxLoad ? 'FULL' : 'OK' }}
              </div>
            </div>
          </div>

          <!-- 路由决策 -->
          <div class="mt-2 pt-2 border-t border-purple-500/20 text-xs space-y-1">
            <div class="flex justify-between">
              <span class="text-purple-300/60">Routing</span>
              <span class="text-purple-200 font-mono">{{ lbStats.currentRouting || '-' }}</span>
            </div>
            <div class="text-purple-400/80 text-[10px] leading-tight min-h-[2em]">
              {{ lbStats.lastDecision }}
            </div>
          </div>
        </div>

        <!-- ===== SERVERS ===== -->
        <div class="absolute right-6 top-4 flex flex-col gap-5">
          <div v-for="s in servers" :key="s.id" class="flex items-start gap-3">
            <div 
              :ref="el => (serverRefs[s.id] = el as HTMLElement)"
              class="border-2 rounded-2xl w-32 backdrop-blur p-3 shadow-lg transition-all duration-300"
              :class="[
                getLoadColor(serverLoad[s.id], s.maxLoad).border,
                getLoadColor(serverLoad[s.id], s.maxLoad).card,
                getLoadColor(serverLoad[s.id], s.maxLoad).glow,
                { 'animate-pulse': serverLoad[s.id] > s.maxLoad }
              ]"
            >
              <div class="text-sky-100 font-bold text-sm text-center">{{ s.label }}</div>
              <div 
                class="text-xs text-center mt-1 font-semibold transition-colors"
                :class="getLoadColor(serverLoad[s.id], s.maxLoad).text"
              >
                {{ getLoadColor(serverLoad[s.id], s.maxLoad).status }}
              </div>
              <div class="mt-2 flex justify-center gap-1">
                <div 
                  v-for="i in s.maxLoad" :key="i" 
                  class="w-3 h-3 rounded transition-all duration-300"
                  :class="i <= serverLoad[s.id] ? getLoadColor(serverLoad[s.id], s.maxLoad).bg : 'bg-gray-600/50'"
                ></div>
              </div>
              <!-- Processing 指示 -->
              <div 
                v-if="activeRequests.some(r => r.serverId === s.id && r.phase === 'proc')"
                class="mt-2 text-center text-xs text-cyan-400 animate-pulse"
              >
                ⚡ Processing...
              </div>
            </div>

            <div class="flex flex-col gap-2">
              <div class="flex gap-1 mb-1">
                <div v-for="m in s.models" :key="m" class="bg-emerald-400 text-black px-2 py-0.5 text-xs font-bold rounded shadow">{{ m }}</div>
              </div>
              <div 
                v-for="k in s.gpus" 
                :key="s.id + 'gpu' + k"
                class="relative bg-gradient-to-r from-gray-900 to-gray-800 border w-56 h-14 rounded flex items-center px-3 shadow-lg transition-all duration-300"
                :class="getLoadColor(serverLoad[s.id], s.maxLoad).border"
              >
                <div class="flex-1">
                  <div class="text-white font-mono text-xs">GPU {{ k }}</div>
                  <div class="text-xs transition-colors" :class="getLoadColor(serverLoad[s.id], s.maxLoad).text">
                    {{ getLoadColor(serverLoad[s.id], s.maxLoad).status }}
                  </div>
                </div>
                <div class="flex flex-col gap-1">
                  <div class="flex gap-0.5">
                    <div 
                      v-for="i in 6" :key="i" 
                      class="w-1.5 h-1.5 rounded-sm transition-all duration-300"
                      :class="[
                        getLoadColor(serverLoad[s.id], s.maxLoad).bg, 
                        serverLoad[s.id] >= s.maxLoad ? 'animate-pulse' : 'opacity-50'
                      ]"
                    ></div>
                  </div>
                  <div class="flex gap-0.5">
                    <div v-for="i in 6" :key="i" class="w-1.5 h-1.5 rounded-sm bg-gray-600/40"></div>
                  </div>
                </div>
                <div class="absolute bottom-1 left-3 right-3 h-1.5 bg-gray-700 rounded overflow-hidden">
                  <div 
                    class="h-full rounded transition-all duration-500"
                    :class="getLoadColor(serverLoad[s.id], s.maxLoad).bg"
                    :style="{ width: `${getGpuPercent(serverLoad[s.id], s.maxLoad)}%` }"
                  ></div>
                </div>
              </div>
            </div>
          </div>
          <div class="text-sky-400 font-bold text-center tracking-widest text-xs">TRITON INFERENCE SERVERS</div>
        </div>

        <!-- ===== MONITORING ===== -->
        <div 
          ref="monitorRef"
          class="absolute bottom-5 left-1/2 -translate-x-1/2 w-[480px] border rounded-xl backdrop-blur p-4 shadow-lg transition-all duration-300"
          :class="totalLoad >= 4 ? 'border-red-400/70 bg-red-900/20' : totalLoad >= 3 ? 'border-orange-400/50 bg-orange-900/10' : 'border-rose-400/40 bg-rose-900/10'"
        >
          <div class="flex items-center gap-3 mb-3">
            <div class="text-xl">📊</div>
            <div>
              <div class="text-rose-100 font-bold text-sm">Monitoring & Metrics</div>
              <div class="text-rose-400/60 text-xs">Real-time Load Balancing</div>
            </div>
            <div class="ml-auto flex items-center gap-2">
              <div 
                v-if="totalLoad >= 3" 
                class="flex items-center gap-1 px-2 py-0.5 rounded text-xs animate-pulse"
                :class="totalLoad >= 4 ? 'bg-red-500/30 text-red-400' : 'bg-orange-500/20 text-orange-400'"
              >
                <span>{{ totalLoad >= 4 ? '🔥' : '⚠️' }}</span> 
                {{ totalLoad >= 4 ? 'HIGH LOAD' : 'BUSY' }}
              </div>
              <div class="flex items-center gap-1">
                <div class="w-2 h-2 bg-green-400 rounded-full animate-pulse"></div>
                <span class="text-green-400 text-xs">Live</span>
              </div>
            </div>
          </div>
          <div class="grid grid-cols-6 gap-2 text-center">
            <div class="bg-black/30 rounded p-2">
              <div class="text-rose-300 text-xs">Latency</div>
              <div class="font-mono text-sm" :class="avgLatency > 30 ? 'text-red-400' : avgLatency > 20 ? 'text-yellow-400' : 'text-white'">
                {{ avgLatency }}ms
              </div>
            </div>
            <div class="bg-black/30 rounded p-2">
              <div class="text-rose-300 text-xs">Active</div>
              <div class="text-white font-mono text-sm">{{ activeRequests.length }}</div>
            </div>
            <div class="bg-black/30 rounded p-2">
              <div class="text-rose-300 text-xs">Total</div>
              <div class="text-white font-mono text-sm">{{ lbStats.total }}</div>
            </div>
            <div class="bg-black/30 rounded p-2">
              <div class="text-rose-300 text-xs">S1 Load</div>
              <div class="font-mono text-sm" :class="getLoadColor(serverLoad.s1, 2).text">
                {{ serverLoad.s1 }}/2
              </div>
            </div>
            <div class="bg-black/30 rounded p-2">
              <div class="text-rose-300 text-xs">S2 Load</div>
              <div class="font-mono text-sm" :class="getLoadColor(serverLoad.s2, 2).text">
                {{ serverLoad.s2 }}/2
              </div>
            </div>
            <div class="bg-black/30 rounded p-2">
              <div class="text-rose-300 text-xs">Queue</div>
              <div class="font-mono text-sm" :class="totalLoad > 3 ? 'text-red-400' : 'text-white'">
                {{ Math.max(0, totalLoad - 4) }}
              </div>
            </div>
          </div>
        </div>

        <!-- ===== SVG ===== -->
        <svg class="absolute inset-0 w-full h-full pointer-events-none" style="overflow: visible;">
          <defs>
            <linearGradient id="resGradFull2" x1="0%" y1="0%" x2="100%" y2="0%">
              <stop offset="0%" stop-color="#34d399" />
              <stop offset="100%" stop-color="#10b981" />
            </linearGradient>
            <filter id="glowStrong" x="-100%" y="-100%" width="300%" height="300%">
              <feGaussianBlur stdDeviation="4" result="blur"/>
              <feMerge><feMergeNode in="blur"/><feMergeNode in="SourceGraphic"/></feMerge>
            </filter>
          </defs>

          <!-- Monitor lines -->
          <g fill="none" stroke="#f43f5e" stroke-width="1" stroke-dasharray="4 4" opacity="0.15">
            <path v-for="(p, i) in paths.monitorLines" :key="'mon-'+i" :d="p" />
          </g>

          <!-- Background paths - Request -->
          <g fill="none" stroke="#fbbf24" stroke-width="1.5" stroke-dasharray="6 4" opacity="0.12">
            <path v-for="c in clients" :key="'bg-ce-'+c.id" :d="paths.clientToEntry[c.id] || ''" />
          </g>
          <path :d="paths.entryToLb" fill="none" stroke="#a855f7" stroke-width="2" stroke-dasharray="6 4" opacity="0.15" />
          <g fill="none" stroke="#38bdf8" stroke-width="1.5" stroke-dasharray="6 4" opacity="0.12">
            <path v-for="s in servers" :key="'bg-ls-'+s.id" :d="paths.lbToServer[s.id] || ''" />
          </g>

          <!-- Background paths - Response -->
          <g fill="none" stroke="#34d399" stroke-width="1.5" stroke-dasharray="4 4" opacity="0.12">
            <path v-for="s in servers" :key="'bg-sl-'+s.id" :d="paths.serverToLb[s.id] || ''" />
          </g>
          <path :d="paths.lbToEntry" fill="none" stroke="#34d399" stroke-width="2" stroke-dasharray="4 4" opacity="0.12" />
          <g fill="none" stroke="#34d399" stroke-width="1.5" stroke-dasharray="4 4" opacity="0.12">
            <path v-for="c in clients" :key="'bg-ec-'+c.id" :d="paths.entryToClient[c.id] || ''" />
          </g>

          <!-- Active request packets -->
          <g v-for="req in activeRequests" :key="req.id">
            <circle 
              v-if="req.phase !== 'proc' && req.phase !== 'done'"
              :cx="req.x" 
              :cy="req.y" 
              r="6"
              :fill="isRequestPhase(req.phase) ? req.color : 'url(#resGradFull2)'"
              filter="url(#glowStrong)"
            />
            <!-- Packet trail -->
            <circle 
              v-if="req.phase !== 'proc' && req.phase !== 'done'"
              :cx="req.x" 
              :cy="req.y" 
              r="10"
              :fill="isRequestPhase(req.phase) ? req.color : '#34d399'"
              opacity="0.2"
            />
          </g>
        </svg>

        <!-- ===== LEGEND ===== -->
        <div class="absolute top-4 left-6 flex flex-wrap gap-3 bg-black/50 px-4 py-2 rounded-lg border border-white/10">
          <div class="flex items-center gap-2">
            <div class="w-3 h-3 rounded-full bg-amber-400 shadow shadow-amber-400/50"></div>
            <span class="text-xs text-gray-300">Request</span>
          </div>
          <div class="flex items-center gap-2">
            <div class="w-3 h-3 rounded-full bg-emerald-400 shadow shadow-emerald-400/50"></div>
            <span class="text-xs text-gray-300">Response</span>
          </div>
          <div class="flex items-center gap-1">
            <div class="w-2 h-2 rounded bg-emerald-500"></div>
            <div class="w-2 h-2 rounded bg-yellow-500"></div>
            <div class="w-2 h-2 rounded bg-orange-500"></div>
            <div class="w-2 h-2 rounded bg-red-500"></div>
            <span class="text-xs text-gray-300 ml-1">Server Load</span>
          </div>
        </div>

        <div class="absolute top-4 right-6 text-right">
          <div class="text-white/80 text-xs font-mono">NVIDIA Triton</div>
          <div class="text-cyan-400 text-sm font-bold">Intelligent Load Balancing</div>
        </div>
      </div>
    </div>
  </div>
</template>