<script setup lang="ts">
import { onMounted, onBeforeUnmount, reactive, ref, computed } from 'vue'

const props = defineProps<{
  scale?: number
}>()

const scale = computed(() => props.scale ?? 1)

const stage = ref<HTMLElement | null>(null)

// 基础尺寸
const BASE_WIDTH = 1100
const BASE_HEIGHT = 520

// 数据定义
const clients = [{ id: 'c1', label: 'Client A' }, { id: 'c2', label: 'Client B' }, { id: 'c3', label: 'Client C' }, { id: 'c4', label: 'Client D' }]
const servers = [
  { id: 's1', label: 'Triton Server 1', models: ['GNN4ITk', 'Traccc'], coprocessors: 2 },
  { id: 's2', label: 'Triton Server 2', models: ['ExaTrkX', 'Traccc'], coprocessors: 2 }
]

// 连接定义：client -> entry -> lb -> server -> 返回
const clientToEntry = [
  { id: 'ce1', from: 'c1', start: '0s', dur: '0.6s', loopGap: '4s' },
  { id: 'ce2', from: 'c2', start: '1.2s', dur: '0.6s', loopGap: '4.5s' },
  { id: 'ce3', from: 'c3', start: '2.4s', dur: '0.6s', loopGap: '5s' }
]

const entryToLb = [
  { id: 'el1', trigger: 'ce1', dur: '0.4s', delay: '0.1s' },
  { id: 'el2', trigger: 'ce2', dur: '0.4s', delay: '0.1s' },
  { id: 'el3', trigger: 'ce3', dur: '0.4s', delay: '0.1s' }
]

const lbToServer = [
  { id: 'ls1', trigger: 'el1', to: 's1', dur: '0.5s', delay: '0.15s', bend: 0.15 },
  { id: 'ls2', trigger: 'el2', to: 's2', dur: '0.5s', delay: '0.15s', bend: 0.12 },
  { id: 'ls3', trigger: 'el3', to: 's1', dur: '0.5s', delay: '0.15s', bend: 0.18 }
]

// Refs
const clientRefs: Record<string, HTMLElement | null> = reactive({})
const entryRef = ref<HTMLElement | null>(null)
const lbRef = ref<HTMLElement | null>(null)
const serverRefs: Record<string, HTMLElement | null> = reactive({})
const monitorRef = ref<HTMLElement | null>(null)

// Paths
const paths = reactive({
  clientToEntry: {} as Record<string, string>,
  entryToLb: '',
  lbToServer: {} as Record<string, { req: string; res: string }>,
  monitorLines: [] as string[]
})

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

function center(el: HTMLElement, root: HTMLElement) {
  const r = getRelativeRect(el, root)
  return { x: r.left + r.width / 2, y: r.top + r.height / 2 }
}

function line(from: { x: number; y: number }, to: { x: number; y: number }) {
  return `M ${from.x.toFixed(1)} ${from.y.toFixed(1)} L ${to.x.toFixed(1)} ${to.y.toFixed(1)}`
}

function quad(from: { x: number; y: number }, to: { x: number; y: number }, bend = 0.12, sign = -1) {
  const dx = to.x - from.x
  const cx = from.x + dx * 0.5
  const cy = (from.y + to.y) * 0.5 + sign * Math.abs(dx) * bend
  return `M ${from.x.toFixed(1)} ${from.y.toFixed(1)} Q ${cx.toFixed(1)} ${cy.toFixed(1)} ${to.x.toFixed(1)} ${to.y.toFixed(1)}`
}

function updatePaths() {
  if (!stage.value || !entryRef.value || !lbRef.value) return
  const root = stage.value

  // Client -> Entry
  for (const c of clientToEntry) {
    const cEl = clientRefs[c.from]
    if (!cEl) continue
    const A = rightCenter(cEl, root)
    const B = leftCenter(entryRef.value, root)
    paths.clientToEntry[c.id] = quad(A, B, 0.08, c.from === 'c2' ? 0 : (c.from === 'c1' ? -1 : 1))
  }

  // Entry -> LB
  const entryRight = rightCenter(entryRef.value, root)
  const lbLeft = leftCenter(lbRef.value, root)
  paths.entryToLb = line(entryRight, lbLeft)

  // LB -> Server
  for (const ls of lbToServer) {
    const sEl = serverRefs[ls.to]
    if (!sEl) continue
    const A = rightCenter(lbRef.value, root)
    const B = leftCenter(sEl, root)
    const dReq = quad(A, B, ls.bend, ls.to === 's1' ? -1 : 1)
    const dRes = quad(B, A, ls.bend * 0.6, ls.to === 's1' ? 1 : -1)
    paths.lbToServer[ls.id] = { req: dReq, res: dRes }
  }

  // Monitor 连线
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

let ro: ResizeObserver | null = null

onMounted(() => {
  requestAnimationFrame(() => updatePaths())
  ro = new ResizeObserver(() => requestAnimationFrame(updatePaths))
  if (stage.value) ro.observe(stage.value)
  window.addEventListener('resize', updatePaths, { passive: true })
})

onBeforeUnmount(() => {
  if (ro && stage.value) ro.unobserve(stage.value)
  ro = null
  window.removeEventListener('resize', updatePaths)
})
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
        <div class="absolute left-6 top-16 flex flex-col gap-6 w-36">
          <div 
            v-for="c in clients" 
            :key="c.id"
            :ref="el => (clientRefs[c.id] = el as HTMLElement)"
            class="relative border-2 border-cyan-400 rounded-xl px-4 py-3 text-center bg-cyan-900/20 backdrop-blur shadow-lg shadow-cyan-500/10"
          >
            <div class="text-cyan-100 font-bold text-sm">{{ c.label }}</div>
            <div class="text-cyan-400/60 text-xs mt-1">gRPC Client</div>
            <!-- 小脉冲动画 -->
            <div class="absolute -right-1 -top-1 w-2 h-2 bg-green-400 rounded-full animate-pulse"></div>
          </div>
          <div class="text-cyan-400 font-bold text-center tracking-widest text-xs mt-2">CLIENTS</div>
        </div>

        <!-- ===== NETWORK ENTRY POINT ===== -->
        <div 
          ref="entryRef"
          class="absolute left-52 top-28 w-32 border-2 border-amber-400 rounded-2xl bg-amber-900/20 backdrop-blur p-4 shadow-lg shadow-amber-500/10"
        >
          <div class="text-center">
            <div class="text-amber-300 text-2xl mb-2">🌐</div>
            <div class="text-amber-100 font-bold text-sm">Entry Point</div>
            <div class="text-amber-400/60 text-xs mt-1">API Gateway</div>
          </div>
          <!-- 流量指示器 -->
          <div class="mt-3 flex justify-center gap-1">
            <div class="w-1.5 h-4 bg-amber-400/40 rounded-full animate-pulse" style="animation-delay: 0s;"></div>
            <div class="w-1.5 h-6 bg-amber-400/60 rounded-full animate-pulse" style="animation-delay: 0.2s;"></div>
            <div class="w-1.5 h-5 bg-amber-400/50 rounded-full animate-pulse" style="animation-delay: 0.4s;"></div>
            <div class="w-1.5 h-7 bg-amber-400/70 rounded-full animate-pulse" style="animation-delay: 0.6s;"></div>
          </div>
        </div>

        <!-- ===== LOAD BALANCER ===== -->
        <div 
          ref="lbRef"
          class="absolute left-96 top-24 w-40 border-2 border-purple-400 rounded-2xl bg-purple-900/20 backdrop-blur p-4 shadow-lg shadow-purple-500/20"
        >
          <div class="text-center">
            <div class="text-purple-300 text-2xl mb-2">⚖️</div>
            <div class="text-purple-100 font-bold text-sm">Load Balancer</div>
            <div class="text-purple-400/60 text-xs mt-1">Round Robin</div>
          </div>
          <!-- 分发指示器 -->
          <div class="mt-3 grid grid-cols-2 gap-2">
            <div class="text-center">
              <div class="text-green-400 text-xs font-mono">S1</div>
              <div class="h-1.5 bg-green-400/50 rounded mt-1 animate-pulse"></div>
            </div>
            <div class="text-center">
              <div class="text-blue-400 text-xs font-mono">S2</div>
              <div class="h-1.5 bg-blue-400/50 rounded mt-1 animate-pulse" style="animation-delay: 0.5s;"></div>
            </div>
          </div>
          <!-- 连接数 -->
          <div class="mt-2 text-center text-purple-300/80 text-xs font-mono">
            Active: <span class="text-green-400">3</span>
          </div>
        </div>

        <!-- ===== TRITON SERVERS ===== -->
        <div class="absolute right-6 top-8 flex flex-col gap-8">
          <div v-for="s in servers" :key="s.id" class="flex items-start gap-3">
            <!-- Server CPU -->
            <div 
              :ref="el => (serverRefs[s.id] = el as HTMLElement)"
              class="border-2 border-sky-400 rounded-2xl w-32 bg-sky-900/20 backdrop-blur p-3 shadow-lg shadow-sky-500/10"
            >
              <div class="text-sky-100 font-bold text-sm text-center">{{ s.label }}</div>
              <div class="text-sky-400/60 text-xs text-center mt-1">CPUs</div>
              <div class="mt-2 flex justify-center gap-1">
                <div v-for="i in 4" :key="i" class="w-2 h-2 bg-sky-400/50 rounded-sm"></div>
              </div>
            </div>

            <!-- Coprocessors -->
            <div class="flex flex-col gap-2">
              <!-- Models -->
              <div class="flex gap-1 mb-1">
                <div 
                  v-for="m in s.models" 
                  :key="m"
                  class="bg-emerald-400 text-black px-2 py-0.5 text-xs font-bold rounded shadow"
                >
                  {{ m }}
                </div>
              </div>
              <!-- GPU Cards -->
              <div 
                v-for="k in s.coprocessors" 
                :key="s.id + 'gpu' + k"
                class="relative bg-gradient-to-r from-gray-900 to-gray-800 border border-sky-700 w-56 h-14 rounded flex items-center px-3 shadow-lg"
              >
                <div class="flex-1">
                  <div class="text-white font-mono text-xs">GPU {{ k }}</div>
                  <div class="text-sky-400/60 text-xs">NVIDIA A100</div>
                </div>
                <div class="flex flex-col gap-1">
                  <div class="flex gap-0.5">
                    <div v-for="i in 6" :key="i" class="w-1.5 h-1.5 bg-sky-500/60 rounded-sm"></div>
                  </div>
                  <div class="flex gap-0.5">
                    <div v-for="i in 6" :key="i" class="w-1.5 h-1.5 bg-sky-500/40 rounded-sm"></div>
                  </div>
                </div>
                <!-- 使用率条 -->
                <div class="absolute bottom-1 left-3 right-3 h-1 bg-gray-700 rounded overflow-hidden">
                  <div 
                    class="h-full bg-gradient-to-r from-green-500 to-emerald-400 rounded transition-all"
                    :style="{ width: `${50 + Math.random() * 40}%` }"
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
          class="absolute bottom-6 left-1/2 -translate-x-1/2 w-80 border border-rose-400/50 rounded-xl bg-rose-900/10 backdrop-blur p-4 shadow-lg"
        >
          <div class="flex items-center gap-3 mb-3">
            <div class="text-rose-400 text-xl">📊</div>
            <div>
              <div class="text-rose-100 font-bold text-sm">Monitoring & Metrics</div>
              <div class="text-rose-400/60 text-xs">Prometheus + Grafana</div>
            </div>
            <div class="ml-auto flex items-center gap-1">
              <div class="w-2 h-2 bg-green-400 rounded-full animate-pulse"></div>
              <span class="text-green-400 text-xs">Live</span>
            </div>
          </div>
          <div class="grid grid-cols-4 gap-2 text-center">
            <div class="bg-black/30 rounded p-2">
              <div class="text-rose-300 text-xs">Latency</div>
              <div class="text-white font-mono text-sm">12ms</div>
            </div>
            <div class="bg-black/30 rounded p-2">
              <div class="text-rose-300 text-xs">RPS</div>
              <div class="text-white font-mono text-sm">1.2k</div>
            </div>
            <div class="bg-black/30 rounded p-2">
              <div class="text-rose-300 text-xs">GPU Util</div>
              <div class="text-white font-mono text-sm">78%</div>
            </div>
            <div class="bg-black/30 rounded p-2">
              <div class="text-rose-300 text-xs">Queue</div>
              <div class="text-white font-mono text-sm">3</div>
            </div>
          </div>
        </div>

        <!-- ===== SVG PATHS ===== -->
        <svg class="absolute inset-0 w-full h-full pointer-events-none" style="overflow: visible;">
          <defs>
            <!-- 渐变定义 -->
            <linearGradient id="reqGrad" x1="0%" y1="0%" x2="100%" y2="0%">
              <stop offset="0%" stop-color="#fbbf24" />
              <stop offset="100%" stop-color="#f59e0b" />
            </linearGradient>
            <linearGradient id="resGrad" x1="0%" y1="0%" x2="100%" y2="0%">
              <stop offset="0%" stop-color="#34d399" />
              <stop offset="100%" stop-color="#10b981" />
            </linearGradient>
            <!-- 发光滤镜 -->
            <filter id="glow">
              <feGaussianBlur stdDeviation="2" result="coloredBlur"/>
              <feMerge>
                <feMergeNode in="coloredBlur"/>
                <feMergeNode in="SourceGraphic"/>
              </feMerge>
            </filter>
          </defs>

          <!-- Monitor 连线（虚线） -->
          <g fill="none" stroke="#f43f5e" stroke-width="1" stroke-dasharray="4 4" opacity="0.3">
            <path v-for="(p, i) in paths.monitorLines" :key="'mon-'+i" :d="p" />
          </g>

          <!-- Client -> Entry 背景线 -->
          <g fill="none" stroke="#fbbf24" stroke-width="1.5" stroke-dasharray="6 4" opacity="0.2">
            <path v-for="c in clientToEntry" :key="'ce-bg-'+c.id" :d="paths.clientToEntry[c.id] || ''" />
          </g>

          <!-- Entry -> LB 背景线 -->
          <path :d="paths.entryToLb" fill="none" stroke="#a855f7" stroke-width="2" stroke-dasharray="6 4" opacity="0.3" />

          <!-- LB -> Server 背景线 -->
          <g fill="none" stroke="#38bdf8" stroke-width="1.5" stroke-dasharray="6 4" opacity="0.2">
            <path v-for="ls in lbToServer" :key="'ls-bg-'+ls.id" :d="paths.lbToServer[ls.id]?.req || ''" />
          </g>
          <g fill="none" stroke="#34d399" stroke-width="1.5" stroke-dasharray="4 4" opacity="0.2">
            <path v-for="ls in lbToServer" :key="'ls-res-bg-'+ls.id" :d="paths.lbToServer[ls.id]?.res || ''" />
          </g>

          <!-- Path definitions for animations -->
          <defs>
            <path v-for="c in clientToEntry" :key="'def-ce-'+c.id" :id="'p_ce_'+c.id" :d="paths.clientToEntry[c.id] || ''" />
            <path id="p_el" :d="paths.entryToLb" />
            <path v-for="ls in lbToServer" :key="'def-ls-'+ls.id" :id="'p_ls_'+ls.id" :d="paths.lbToServer[ls.id]?.req || ''" />
            <path v-for="ls in lbToServer" :key="'def-ls-res-'+ls.id" :id="'p_ls_res_'+ls.id" :d="paths.lbToServer[ls.id]?.res || ''" />
          </defs>

          <!-- ===== 动画圆点 ===== -->
          <!-- Client -> Entry -->
          <g v-for="(c, idx) in clientToEntry" :key="'anim-ce-'+c.id">
            <circle r="5" fill="url(#reqGrad)" filter="url(#glow)" opacity="0">
              <animateMotion 
                :id="'ce_'+c.id" 
                :dur="c.dur" 
                :begin="`${c.start}; ls_res_${lbToServer[idx].id}.end + ${c.loopGap}`"
                fill="freeze"
              >
                <mpath :href="'#p_ce_'+c.id" />
              </animateMotion>
              <animate attributeName="opacity" values="0;1;1;0" keyTimes="0;0.1;0.9;1" :dur="c.dur" :begin="`${c.start}; ls_res_${lbToServer[idx].id}.end + ${c.loopGap}`" />
            </circle>
          </g>

          <!-- Entry -> LB -->
          <g v-for="(e, idx) in entryToLb" :key="'anim-el-'+e.id">
            <circle r="5" fill="#a855f7" filter="url(#glow)" opacity="0">
              <animateMotion 
                :id="'el_'+e.id" 
                :dur="e.dur" 
                :begin="`ce_${clientToEntry[idx].id}.end + ${e.delay}`"
                fill="freeze"
              >
                <mpath href="#p_el" />
              </animateMotion>
              <animate attributeName="opacity" values="0;1;1;0" keyTimes="0;0.1;0.9;1" :dur="e.dur" :begin="`ce_${clientToEntry[idx].id}.end + ${e.delay}`" />
            </circle>
          </g>

          <!-- LB -> Server (Request) -->
          <g v-for="(ls, idx) in lbToServer" :key="'anim-ls-'+ls.id">
            <circle r="5" fill="#38bdf8" filter="url(#glow)" opacity="0">
              <animateMotion 
                :id="'ls_'+ls.id" 
                :dur="ls.dur" 
                :begin="`el_${entryToLb[idx].id}.end + ${ls.delay}`"
                fill="freeze"
              >
                <mpath :href="'#p_ls_'+ls.id" />
              </animateMotion>
              <animate attributeName="opacity" values="0;1;1;0" keyTimes="0;0.1;0.9;1" :dur="ls.dur" :begin="`el_${entryToLb[idx].id}.end + ${ls.delay}`" />
            </circle>
          </g>

          <!-- Server -> LB (Response) -->
          <g v-for="(ls, idx) in lbToServer" :key="'anim-ls-res-'+ls.id">
            <circle r="5" fill="url(#resGrad)" filter="url(#glow)" opacity="0">
              <animateMotion 
                :id="'ls_res_'+ls.id" 
                :dur="ls.dur" 
                :begin="`ls_${ls.id}.end + 0.3s`"
                fill="freeze"
              >
                <mpath :href="'#p_ls_res_'+ls.id" />
              </animateMotion>
              <animate attributeName="opacity" values="0;1;1;0" keyTimes="0;0.1;0.9;1" :dur="ls.dur" :begin="`ls_${ls.id}.end + 0.3s`" />
            </circle>
          </g>
        </svg>

        <!-- ===== LEGEND ===== -->
        <div class="absolute top-4 left-6 flex gap-4 bg-black/40 px-4 py-2 rounded-lg border border-white/10">
          <div class="flex items-center gap-2">
            <div class="w-3 h-3 rounded-full bg-amber-400 shadow-lg shadow-amber-400/50"></div>
            <span class="text-xs text-gray-300 font-mono">Request</span>
          </div>
          <div class="flex items-center gap-2">
            <div class="w-3 h-3 rounded-full bg-emerald-400 shadow-lg shadow-emerald-400/50"></div>
            <span class="text-xs text-gray-300 font-mono">Response</span>
          </div>
          <div class="flex items-center gap-2">
            <div class="w-3 h-0.5 bg-rose-400/50"></div>
            <span class="text-xs text-gray-300 font-mono">Metrics</span>
          </div>
        </div>

        <!-- ===== TITLE ===== -->
        <div class="absolute top-4 right-6 text-right">
          <div class="text-white/80 text-xs font-mono">NVIDIA Triton</div>
          <div class="text-cyan-400 text-sm font-bold">Inference Pipeline</div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
@keyframes pulse {
  0%, 100% { opacity: 0.4; }
  50% { opacity: 1; }
}
</style>