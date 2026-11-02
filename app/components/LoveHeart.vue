<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref } from 'vue'

const canvas = ref<HTMLCanvasElement | null>(null)
const started = ref(false)
let startAt = -1

interface Word {
  tx: number
  ty: number
  size: number
  rot: number
  ci: number
  delay: number
  phase: number
}

const COLORS = ['#a855f7', '#c084fc', '#7c3aed', '#d8b4fe', '#ffffff', '#ede9fe']
const TEXT = 'I love you'
const BG = '#000'
const INTRO = 0.8
const BASE = 40 

function inHeart(x: number, y: number) {
  const a = x * x + y * y - 1
  return a * a * a - x * x * y * y * y <= 0
}

function makeWords(count: number): Word[] {
  const list: Word[] = []
  while (list.length < count) {
    const x = Math.random() * 3 - 1.5
    const y = Math.random() * 2.8 - 1.3
    if (!inHeart(x, y)) continue
    list.push({
      tx: x,
      ty: -y,
      size: 9 + Math.random() * 11,
      rot: (Math.random() - 0.5) * 0.9,
      ci: Math.floor(Math.random() * COLORS.length),
      delay: Math.hypot(x, y) * 0.9 + Math.random() * 0.5,
      phase: Math.random() * Math.PI * 2
    })
  }
  return list
}

interface Sprite { c: HTMLCanvasElement, w: number, h: number }

function makeSprites(dpr: number): Sprite[] {
  return COLORS.map((color) => {
    const c = document.createElement('canvas')
    const g = c.getContext('2d')!
    const font = `600 ${BASE * dpr}px system-ui, sans-serif`
    g.font = font
    const pad = 14 * dpr
    const w = Math.ceil(g.measureText(TEXT).width + pad * 2)
    const h = Math.ceil(BASE * dpr * 1.4 + pad * 2)
    c.width = w
    c.height = h
    g.font = font
    g.textAlign = 'center'
    g.textBaseline = 'middle'
    g.shadowColor = color
    g.shadowBlur = 10 * dpr
    g.fillStyle = color
    g.fillText(TEXT, w / 2, h / 2)
    return { c, w: w / dpr, h: h / dpr }
  })
}

function makeGlow(S: number) {
  const r = Math.ceil(S * 2.2)
  const c = document.createElement('canvas')
  c.width = c.height = r * 2
  const g = c.getContext('2d')!
  const grad = g.createRadialGradient(r, r, 0, r, r, r)
  grad.addColorStop(0, 'rgba(168, 85, 247, 0.6)')
  grad.addColorStop(0.45, 'rgba(109, 40, 217, 0.28)')
  grad.addColorStop(1, 'rgba(109, 40, 217, 0)')
  g.fillStyle = grad
  g.fillRect(0, 0, r * 2, r * 2)
  return { c, r }
}

const easeOutCubic = (p: number) => 1 - Math.pow(1 - p, 3)
const clamp01 = (v: number) => Math.min(1, Math.max(0, v))

function start() {
  if (started.value) return
  startAt = performance.now()
  started.value = true
}

let raf = 0
let resizeTimer = 0
let onResize: (() => void) | null = null

onMounted(() => {
  const cv = canvas.value!
  const ctx = cv.getContext('2d', { alpha: false })!
  let W = 0
  let H = 0
  let dpr = 1
  let S = 1
  let sprites: Sprite[] = []
  let glow = makeGlow(1)

  const count = Math.round(Math.min(340, Math.max(160, (window.innerWidth * window.innerHeight) / 4500)))
  const words = makeWords(count)

  const setup = () => {
    const newDpr = Math.min(window.devicePixelRatio || 1, 2)
    W = window.innerWidth
    H = window.innerHeight
    cv.width = Math.round(W * newDpr)
    cv.height = Math.round(H * newDpr)
    cv.style.width = W + 'px'
    cv.style.height = H + 'px'
    S = Math.min(W, H) * 0.3
    glow = makeGlow(S)
    if (newDpr !== dpr || !sprites.length) {
      dpr = newDpr
      sprites = makeSprites(dpr)
    }
  }
  setup()

  onResize = () => {
    clearTimeout(resizeTimer)
    resizeTimer = window.setTimeout(setup, 120)
  }
  window.addEventListener('resize', onResize)
  window.addEventListener('orientationchange', onResize)

  const frame = (now: number) => {
    ctx.setTransform(1, 0, 0, 1, 0, 0)
    ctx.globalAlpha = 1
    ctx.fillStyle = BG
    ctx.fillRect(0, 0, cv.width, cv.height)

    if (startAt >= 0) {
      const t = (now - startAt) / 1000
      const cx = W / 2
      const cy = H / 2

      const beating = t > INTRO + 3
      const b = beating ? Math.pow(Math.abs(Math.sin(t * Math.PI * 1.1)), 6) : 0
      const heartScale = 1 + 0.07 * b

      const glowPulse = (0.85 + 0.15 * Math.sin(t * 2.2) + b * 0.2) * clamp01(t / 1.2)
      ctx.setTransform(dpr, 0, 0, dpr, 0, 0)
      ctx.globalAlpha = Math.min(1, glowPulse)
      ctx.drawImage(glow.c, cx - glow.r, cy - glow.r)

      const baseY = cy + 0.1 * S
      const fontK = Math.max(S / 150, 0.9) / BASE
      for (let i = 0; i < words.length; i++) {
        const w = words[i]!
        const p = (t - INTRO - w.delay) / 1.4
        if (p <= 0) continue
        const e = easeOutCubic(clamp01(p))
        const wiggle = beating ? Math.sin(t * 2 + w.tx * 5) * 0.03 : 0
        const x = cx + w.tx * S * e * heartScale
        const y = baseY + w.ty * S * e * heartScale
        const a = w.rot * e + wiggle
        const cos = Math.cos(a) * dpr
        const sin = Math.sin(a) * dpr
        const sp = sprites[w.ci]!
        const k = w.size * fontK
        const dw = sp.w * k
        const dh = sp.h * k

        ctx.setTransform(cos, sin, -sin, cos, x * dpr, y * dpr)
        ctx.globalAlpha = e
        ctx.drawImage(sp.c, -dw / 2, -dh / 2, dw, dh)
      }
    }

    raf = requestAnimationFrame(frame)
  }
  raf = requestAnimationFrame(frame)
})

onBeforeUnmount(() => {
  cancelAnimationFrame(raf)
  clearTimeout(resizeTimer)
  if (onResize) {
    window.removeEventListener('resize', onResize)
    window.removeEventListener('orientationchange', onResize)
  }
})
</script>

<template>
  <div class="wrap">
    <canvas ref="canvas" class="heart" />
    <Transition name="fade">
      <button v-if="!started" class="start-btn" type="button" @click="start">
        click
      </button>
    </Transition>
  </div>
</template>

<style scoped>
.wrap {
  position: fixed;
  inset: 0;
  background: #000;
}

.heart {
  display: block;
  position: absolute;
  inset: 0;
}

.start-btn {
  position: absolute;
  left: 50%;
  bottom: calc(12% + env(safe-area-inset-bottom, 0px));
  transform: translateX(-50%);
  padding: clamp(12px, 2.4vmin, 18px) clamp(32px, 8vmin, 56px);
  border: 0;
  border-radius: 999px;
  color: #fff;
  font-size: clamp(16px, 3.4vmin, 22px);
  font-family: system-ui, sans-serif;
  cursor: pointer;
  background: linear-gradient(135deg, #a855f7, #7c3aed);
  box-shadow: 0 0 28px rgba(124, 58, 237, 0.6);
  touch-action: manipulation;
  -webkit-tap-highlight-color: transparent;
}

.start-btn:active {
  transform: translateX(-50%) scale(0.95);
}

.fade-leave-active {
  transition: opacity 0.8s ease, transform 0.8s ease;
}

.fade-leave-to {
  opacity: 0;
  transform: translateX(-50%) scale(0.6);
  pointer-events: none;
}
</style>