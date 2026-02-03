<script setup lang="ts">
import { onMounted, onBeforeUnmount, reactive, ref, computed } from 'vue'

const props = defineProps<{
  scale?: number
}>()

const scale = computed(() => props.scale ?? 1)

const stage = ref<HTMLElement | null>(null)

const clients = [{ id: 'c1' }, { id: 'c2' }, { id: 'c3' }]
const servers = [
  { id: 's1', models: ['Model A1', 'Model A2', 'Model B1'], coprocessors: 1 },
  { id: 's2', models: ['Model C', 'Model D'], coprocessors: 2 }
]
const links = [
  { id: 'l1', from: 'c1', to: 's1', start: '0s', reqDur: '1.0s', resDur: '1.0s', resDelay: '0.45s', loopGap: '1.5s', bend: 0.14 },
  { id: 'l2', from: 'c2', to: 's1', start: '0.8s', reqDur: '1.0s', resDur: '1.0s', resDelay: '0.55s', loopGap: '1.7s', bend: 0.10 },
  { id: 'l3', from: 'c3', to: 's2', start: '1.6s', reqDur: '1.1s', resDur: '1.1s', resDelay: '0.60s', loopGap: '1.9s', bend: 0.12 },
  { id: 'l4', from: 'c1', to: 's2', start: '2.4s', reqDur: '1.1s', resDur: '1.1s', resDelay: '0.55s', loopGap: '2.1s', bend: 0.16 }
]

const clientRefs: Record<string, HTMLElement | null> = reactive({})
const serverCpuRefs: Record<string, HTMLElement | null> = reactive({})
const paths: Record<string, { req: string; res: string }> = reactive({})
const labelPos = ref({ x: 260, y: 260 })

// 基础尺寸
const BASE_WIDTH = 860
const BASE_HEIGHT = 420

function getRelativeRect(el: HTMLElement, root: HTMLElement) {
  let left = 0, top = 0
  let current: HTMLElement | null = el
  
  while (current && current !== root) {
    left += current.offsetLeft
    top += current.offsetTop
    current = current.offsetParent as HTMLElement | null
  }
  
  return {
    left,
    top,
    width: el.offsetWidth,
    height: el.offsetHeight,
    right: left + el.offsetWidth,
    bottom: top + el.offsetHeight
  }
}

function rightCenter(el: HTMLElement, root: HTMLElement) {
  const r = getRelativeRect(el, root)
  return { x: r.right, y: r.top + r.height / 2 }
}

function leftCenter(el: HTMLElement, root: HTMLElement) {
  const r = getRelativeRect(el, root)
  return { x: r.left, y: r.top + r.height / 2 }
}

function quad(from: { x: number; y: number }, to: { x: number; y: number }, bend = 0.12, sign = -1) {
  const dx = to.x - from.x
  const cx = from.x + dx * 0.5
  const cy = (from.y + to.y) * 0.5 + sign * Math.abs(dx) * bend
  return `M ${from.x.toFixed(1)} ${from.y.toFixed(1)} Q ${cx.toFixed(1)} ${cy.toFixed(1)} ${to.x.toFixed(1)} ${to.y.toFixed(1)}`
}

function updatePaths() {
  if (!stage.value) return
  const root = stage.value
  
  for (const L of links) {
    const cEl = clientRefs[L.from]
    const sEl = serverCpuRefs[L.to]
    if (!cEl || !sEl) continue
    
    const A = rightCenter(cEl, root)
    const B = leftCenter(sEl, root)
    
    const dReq = quad(A, B, L.bend, -1)
    const dRes = quad(B, A, Math.max(0.06, L.bend * 0.6), +1)
    
    paths[L.id] = { req: dReq, res: dRes }
  }
  
  const ps = Object.values(paths)
  if (ps.length) {
    const mids = ps.map(p => {
      const m = p.req.match(/M ([\d.]+) ([\d.]+) Q ([\d.]+) ([\d.]+) ([\d.]+) ([\d.]+)/)
      return m ? { x: +m[3], y: +m[4] } : { x: 260, y: 260 }
    })
    labelPos.value = {
      x: mids.reduce((a, b) => a + b.x, 0) / mids.length,
      y: mids.reduce((a, b) => a + b.y, 0) / mids.length
    }
  }
}

let ro: ResizeObserver | null = null

onMounted(() => {
  requestAnimationFrame(() => {
    updatePaths()
  })
  
  ro = new ResizeObserver(() => {
    requestAnimationFrame(updatePaths)
  })
  
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
  <!-- 外层：缩放后的实际尺寸，可被 flex 居中 -->
  <div 
    class="relative"
    :style="{ 
      width: `${BASE_WIDTH * scale}px`, 
      height: `${BASE_HEIGHT * scale}px` 
    }"
  >
    <!-- 缩放层 -->
    <div 
      class="origin-top-left"
      :style="{ transform: `scale(${scale})` }"
    >
      <!-- 内容层：固定基础尺寸 -->
      <div 
        ref="stage" 
        class="relative bg-[#0a192f] p-8 rounded-lg"
        :style="{ width: `${BASE_WIDTH}px`, height: `${BASE_HEIGHT}px` }"
      >
        <!-- Clients - 左边 -->
        <div class="absolute left-6 top-10 flex flex-col gap-8 w-44">
          <div 
            v-for="c in clients" 
            :key="c.id" 
            :ref="el => (clientRefs[c.id] = el as HTMLElement)" 
            class="border-2 border-sky-500 rounded-xl px-4 py-3 text-center font-bold text-sky-100 bg-sky-900/30"
          >
            CPUs
          </div>
          <div class="flex flex-col items-center opacity-60 -mt-2 -mb-2">
            <div v-for="j in 8" :key="'dot-'+j" class="w-1 h-1 bg-white rounded-full my-1"></div>
          </div>
          <div class="absolute bottom--7 left-15 text-sky-400 font-bold mt-1 text-left tracking-widest text-sm">
            Clients
          </div>
        </div>

        <!-- Servers - 右边 -->
        <div class="absolute right-10 top-12 flex flex-col gap-14">
          <div v-for="s in servers" :key="s.id" class="flex items-center">
            <div 
              :ref="el => (serverCpuRefs[s.id] = el as HTMLElement)" 
              class="border-2 border-sky-500 rounded-2xl w-36 h-24 flex items-center justify-center font-bold text-xl text-sky-100 bg-white/5 z-10 backdrop-blur"
            >
              CPUs
            </div>
            <div class="flex flex-col gap-2 ml-2 mr-2">
              <div v-for="k in s.coprocessors" :key="s.id+'bar'+k" class="flex gap-1">
                <div class="w-3 h-2 bg-sky-600/80 rounded-sm"></div>
                <div class="w-3 h-2 bg-sky-600/80 rounded-sm"></div>
                <div class="w-3 h-2 bg-sky-600/80 rounded-sm"></div>
              </div>
            </div>
            <div class="relative">
              <div class="absolute -top-12 left-4 flex gap-2">
                <div 
                  v-for="m in s.models" 
                  :key="s.id+m" 
                  class="bg-[#86efac] text-black px-3 py-1 text-xs font-bold border border-black"
                >
                  {{ m }}
                </div>
              </div>
              <div class="flex flex-col gap-3">
                <div 
                  v-for="k in s.coprocessors" 
                  :key="s.id+'cop'+k" 
                  class="bg-black border-2 border-sky-900 w-72 h-16 flex items-center justify-center text-white font-mono shadow-[0_0_15px_rgba(7,89,133,0.25)]"
                >
                  Coprocessor
                </div>
              </div>
            </div>
          </div>
          <div class="text-sky-400 font-bold text-center tracking-widest text-sm -mt-2">
            Servers (Cloud/HPC/Local)
          </div>
        </div>

        <!-- SVG Paths -->
        <svg class="absolute inset-0 w-full h-full pointer-events-none" style="overflow: visible;">
          <g fill="none" stroke="#d97706" stroke-width="1.6" stroke-dasharray="6 4" opacity="0.22">
            <path v-for="L in links" :key="'bg-'+L.id" :d="paths[L.id]?.req || ''" />
          </g>
          <g fill="none" stroke="#ff4444" stroke-width="1.4" stroke-dasharray="4 4" opacity="0.25">
            <path v-for="L in links" :key="'res-bg-'+L.id" :d="paths[L.id]?.res || ''" />
          </g>
          
          <g v-for="L in links" :key="'anim-'+L.id">
            <circle r="4" fill="#fbbf24" opacity="0">
              <animateMotion 
                :id="'req_'+L.id" 
                :dur="L.reqDur" 
                :begin="`${L.start}; res_${L.id}.end + ${L.loopGap}`" 
                fill="freeze"
              >
                <mpath :href="'#p_req_'+L.id" />
              </animateMotion>
              <animate 
                attributeName="opacity" 
                values="0;1;1;0" 
                keyTimes="0;0.08;0.92;1" 
                :dur="L.reqDur" 
                :begin="`${L.start}; res_${L.id}.end + ${L.loopGap}`" 
                repeatCount="1" 
              />
            </circle>
            
            <circle r="4" fill="#ff4444" opacity="0">
              <animateMotion 
                :id="'res_'+L.id" 
                :dur="L.resDur" 
                :begin="`req_${L.id}.end + ${L.resDelay}`" 
                fill="freeze"
              >
                <mpath :href="'#p_res_'+L.id" />
              </animateMotion>
              <animate 
                attributeName="opacity" 
                values="0;1;1;0" 
                keyTimes="0;0.08;0.92;1" 
                :dur="L.resDur" 
                :begin="`req_${L.id}.end + ${L.resDelay}`" 
                repeatCount="1" 
              />
            </circle>
            
            <defs>
              <path :id="'p_req_'+L.id" :d="paths[L.id]?.req || ''" />
              <path :id="'p_res_'+L.id" :d="paths[L.id]?.res || ''" />
            </defs>
          </g>
          
          <text 
            :x="labelPos.x - 16" 
            :y="labelPos.y" 
            fill="#d97706" 
            font-size="7" 
            font-family="monospace" 
            font-weight="bold" 
            opacity="0.7"
          >
            gRPC
          </text>
        </svg>

        <!-- Legend -->
        <div class="absolute bottom-4 left-50 flex flex-col gap-2 bg-black/40 p-3 rounded border border-white/10">
          <div class="flex items-center gap-3">
            <div class="w-3 h-3 rounded-full bg-[#fbbf24]"></div>
            <span class="text-xs text-gray-300 font-mono">Inference Request</span>
          </div>
          <div class="flex items-center gap-3">
            <div class="w-3 h-3 rounded-full bg-[#ff4444]"></div>
            <span class="text-xs text-gray-300 font-mono">Result Return</span>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>