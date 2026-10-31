<!-- Phase4Finale.vue -->
<template>
  <section
    ref="stageRef"
    class="finale"
    :class="{
      'is-pile': phase === 'pile',
      'is-interactive': phase === 'heart',
      'is-filled': phase === 'shrinking' || phase === 'filled'
    }"
    :style="stageStyle"
  >
    <!-- Efek gelombang kejut saat foto meledak -->
    <div ref="shockRef" class="shock"></div>

    <!-- State 5: 3 video pop-up di sekitar hati (muncul setelah hati selesai merapat) -->
    <template v-if="showVideos">
      <div
        v-for="(v, i) in popupVideos"
        :key="v.id"
        class="pop-video"
        :style="popStyle(i)"
      >
        <video
          :src="v.src"
          :style="videoStyle(i)"
          autoplay
          loop
          muted
          playsinline
          preload="auto"
          @loadedmetadata="onVideoMeta(i, $event)"
        ></video>
      </div>
    </template>

    <!-- 60 foto -->
    <div ref="groupRef" class="heart-group">
      <div v-for="(photo, i) in photos" :key="photo.id" class="photo-wrap">
        <button
          class="photo"
          type="button"
          :tabindex="phase === 'heart' ? 0 : -1"
          :aria-label="`Buka foto ${i + 1}`"
          @click="openModal(photo)"
        >
          <img :src="encodeURI(photo.src)" alt="" draggable="false" @error="onImgError" />
        </button>
      </div>
    </div>

    <!-- State 1: area klik di atas tumpukan foto -->
    <button
      v-if="phase === 'pile'"
      class="pile-hit"
      type="button"
      aria-label="Klik tumpukan foto untuk membuka kenangan"
      @click="explodeToHeart"
    ></button>

    <!-- State 1: petunjuk + 3 anak panah -->
    <Transition name="fade">
      <div v-if="showHint" class="pile-hint">
        <p class="hint-text">Klik bagian tengah ini</p>
        <div class="arrows" aria-hidden="true">
          <svg v-for="n in 3" :key="n" class="arrow" :style="{ animationDelay: `${(n - 1) * 0.18}s` }" viewBox="0 0 24 24">
            <path d="M5 8l7 7 7-7" />
          </svg>
        </div>
      </div>
    </Transition>

    <!-- State 4: zona rahasia di rongga hati -->
    <button
      class="secret-zone"
      :class="{
        'is-ready': phase === 'heart',
        'is-active': orbVisible,
        'is-burst': phase === 'shrinking' || phase === 'filled'
      }"
      type="button"
      aria-label="Kejutan rahasia"
      @pointerenter="onZoneEnter"
      @pointerleave="onZoneLeave"
      @pointerdown="onZonePointerDown"
      @focus="onZoneFocus"
      @blur="onZoneBlur"
      @click="onZoneClick"
    >
      <span class="orb"></span>
    </button>

    <!-- Petunjuk saat hati sudah terbentuk -->
    <Transition name="fade">
      <p v-if="phase === 'heart'" class="hint-bottom">
        Sentuh foto untuk melihat kenangan ✨ &nbsp;·&nbsp; ada kejutan di tengah hati
      </p>
    </Transition>

    <!-- State 6: tombol Back -->
    <Transition name="fade">
      <button v-if="showBack" class="back-btn" type="button" @click="goBack">
        <svg viewBox="0 0 24 24" aria-hidden="true">
          <path d="M15 5l-7 7 7 7" />
        </svg>
        <span>Back</span>
      </button>
    </Transition>

    <!-- State 3: Modal Polaroid -->
    <Transition name="modal">
      <div
        v-if="isModalOpen"
        class="modal"
        role="dialog"
        aria-modal="true"
        aria-label="Foto kenangan"
        @click.self="closeModal"
      >
        <div class="modal-polaroid">
          <div class="modal-photo">
            <img :src="encodeURI(selectedPhoto)" alt="Foto kenangan" draggable="false" @error="onImgError" />
          </div>
          <p class="modal-caption">Happy Birthday MyGembolow💕</p>
          <button class="close-btn" type="button" @click="closeModal">Tutup</button>
        </div>
      </div>
    </Transition>
  </section>
</template>

<script setup>
import { ref, computed, onMounted, onBeforeUnmount, nextTick } from 'vue'
import gsap from 'gsap'

const emit = defineEmits(['back-to-menu'])

/* ================= KONFIGURASI ================= */
const PHOTO_COUNT = 60
const HINT_DELAY_MS = 2000 // jeda sebelum petunjuk "Klik bagian tengah ini" muncul
const BACK_DELAY_MS = 2500 // tombol Back muncul sekian ms setelah video pop-up mulai muncul
const PHOTO_MAX = 85 // ukuran foto di layar lebar
const PHOTO_MIN = 62 // ukuran foto di layar kecil
const BAND = 0.2 // foto berselang-seling sedikit maju/mundur agar garis hati lebih "tebal"
const RINGS = [1, 0.72, 0.46, 0.22] // hati bersarang yang mengisi rongga (State 5)
const SHRINK = 0.62 // ukuran formasi akhir dibanding hati awal
const POP_SIZE = 1.45 // pengali ukuran video pop-up (versi sebelumnya 1.2)
const POP_GAP = 16 // jarak video ke hati (px). Ubah 10 - 20 sesuai selera
const POP_EDGE = 10 // jarak minimum video ke tepi layar (px)

/* ================= DATA ================= */
const photos = ref([])
for (let i = 1; i <= PHOTO_COUNT; i++) {
  photos.value.push({ id: i, src: `/image/naura (${i}).webp` })
}

// 3 video pop-up pendek (looping seperti GIF). Ganti path sesuai file kamu
// Video 1: kanan atas, Video 2: kanan bawah, Video 3: kiri atas
const popupVideos = [
  { id: 1, src: '/video/video (1).mp4' },
  { id: 2, src: '/video/video (2).mp4' },
  { id: 3, src: '/video/video (3).mp4' }
]

// Arah pencarian posisi tiap video (sudut dari garis horizontal, makin besar = makin vertikal)
const POP_SLOTS = [
  { sx: 1, sy: -1, angles: [26, 38, 50, 62, 74] }, // kanan atas
  { sx: 1, sy: 1, angles: [26, 38, 50, 62, 74] }, // kanan bawah
  { sx: -1, sy: -1, angles: [26, 38, 50, 62, 74] } // kiri atas
]

const selectedPhoto = ref('')
const isModalOpen = ref(false)

/* ================= STATE ================= */
// 'pile' -> 'exploding' -> 'heart' -> 'shrinking' -> 'filled'
const phase = ref('pile')
const showHint = ref(false)
const orbVisible = ref(false)
const showVideos = ref(false)
const showBack = ref(false)

const stageRef = ref(null)
const groupRef = ref(null)
const shockRef = ref(null)

let ctx = null
let beatTween = null
let resizeTimer = null
let lastShownAt = 0
let hiddenOnDown = false
const timers = []

/* ================= MATEMATIKA HATI ================= */
/*
  x = 16 sin^3(t)
  y = 13cos(t) - 5cos(2t) - 2cos(3t) - cos(4t)
  Disampel rapat, dipusatkan ke bounding box, lalu dihitung panjang busurnya
  supaya foto tersebar rata sepanjang garis hati.
*/
const heartPath = (() => {
  const STEPS = 720
  const raw = []
  for (let i = 0; i <= STEPS; i++) {
    const t = (i / STEPS) * Math.PI * 2
    raw.push({
      x: 16 * Math.pow(Math.sin(t), 3),
      y: 13 * Math.cos(t) - 5 * Math.cos(2 * t) - 2 * Math.cos(3 * t) - Math.cos(4 * t)
    })
  }
  const ys = raw.map((p) => p.y)
  const minY = Math.min(...ys)
  const maxY = Math.max(...ys)
  const midY = (minY + maxY) / 2
  const pts = raw.map((p) => ({ x: p.x, y: p.y - midY }))
  const cum = [0]
  for (let i = 1; i < pts.length; i++) {
    cum.push(cum[i - 1] + Math.hypot(pts[i].x - pts[i - 1].x, pts[i].y - pts[i - 1].y))
  }
  return { pts, cum, length: cum[cum.length - 1], width: 32, height: maxY - minY }
})()

// Titik pada garis hati berdasarkan fraksi panjang busur (0..1)
function pointAt(frac) {
  const { pts, cum, length } = heartPath
  const target = (((frac % 1) + 1) % 1) * length
  let lo = 0
  let hi = cum.length - 1
  while (hi - lo > 1) {
    const mid = (lo + hi) >> 1
    if (cum[mid] <= target) lo = mid
    else hi = mid
  }
  const seg = cum[hi] - cum[lo] || 1
  const k = (target - cum[lo]) / seg
  return {
    x: pts[lo].x + (pts[hi].x - pts[lo].x) * k,
    y: pts[lo].y + (pts[hi].y - pts[lo].y) * k
  }
}

/* ================= LAYOUT ================= */
const clamp = (v, a, b) => Math.min(b, Math.max(a, v))

const photoSize = ref(PHOTO_MAX)
const zoneSize = ref(120)
const orbSize = ref(64)
const zoneY = ref(0)
const popPad = ref(6) // tebal frame putih video (px)

// Rasio asli tiap video (lebar / tinggi), diperbarui saat metadata video termuat
const popRatios = ref([0.5625, 0.5625, 0.5625])
// Ukuran tampil tiap video (px, tanpa frame) dan posisi pusatnya relatif ke tengah layar
const popSizes = ref([
  { w: 160, h: 284 },
  { w: 160, h: 284 },
  { w: 160, h: 284 }
])
const popPos = ref([
  { x: 0, y: 0 },
  { x: 0, y: 0 },
  { x: 0, y: 0 }
])

let outlinePts = [] // posisi foto membentuk garis hati (State 2)
let fillPts = [] // posisi foto saat merapat memenuhi rongga (State 5)
let outerRing = [] // titik tepi luar hati yang sudah merapat (untuk menjaga jarak video)
let lastSize = PHOTO_MAX

function computeLayout() {
  const vw = window.innerWidth
  const vh = window.innerHeight
  const n = photos.value.length

  // Ukuran dasar foto diperbesar: 62px (HP) sampai 85px (laptop)
  const size = Math.round(clamp(Math.min(vw, vh) * 0.13, PHOTO_MIN, PHOTO_MAX))

  // Skala hati agar muat di layar
  const s = clamp(
    Math.min(
      (vw * 0.94 - size * 1.4) / heartPath.width,
      (vh * 0.86 - size * 1.4) / heartPath.height
    ),
    4,
    30
  )

  photoSize.value = size
  lastSize = size

  // Formasi garis hati (foto selang-seling sedikit ke luar / ke dalam)
  const off = size * BAND
  outlinePts = Array.from({ length: n }, (_, i) => {
    const f = i / n
    const p = pointAt(f)
    const a = pointAt(f - 0.003)
    const b = pointAt(f + 0.003)
    const tx = b.x - a.x
    const ty = b.y - a.y
    const tl = Math.hypot(tx, ty) || 1
    let nx = ty / tl
    let ny = -tx / tl
    if (nx * p.x + ny * p.y < 0) {
      nx = -nx
      ny = -ny
    }
    const o = i % 2 ? off : -off
    return { x: p.x * s + nx * o, y: -(p.y * s + ny * o) }
  })

  // Cari titik pusat rongga yang paling lapang (jarak terjauh dari garis hati)
  const dense = []
  for (let i = 0; i < heartPath.pts.length; i += 6) {
    dense.push({ x: heartPath.pts[i].x * s, y: -heartPath.pts[i].y * s })
  }
  const spanY = heartPath.height * s
  let best = { cy: 0, d: 0 }
  for (let k = -20; k <= 20; k++) {
    const cy = (k / 20) * spanY * 0.3
    let m = Infinity
    for (const q of dense) {
      const d = Math.hypot(q.x, q.y - cy)
      if (d < m) m = d
    }
    if (m > best.d) best = { cy, d: m }
  }
  zoneY.value = Math.round(best.cy)
  zoneSize.value = Math.round(clamp(2 * (best.d - size * (0.5 + BAND)) * 0.9, 56, 240))
  orbSize.value = Math.round(clamp(zoneSize.value * 0.5, 44, 96))

  // Formasi padat: beberapa hati bersarang
  const sum = RINGS.reduce((acc, v) => acc + v, 0)
  const counts = RINGS.map((f) => Math.max(1, Math.round((n * f) / sum)))
  counts[0] += n - counts.reduce((acc, v) => acc + v, 0)

  const targets = []
  RINGS.forEach((f, r) => {
    for (let k = 0; k < counts[r]; k++) {
      const a = (k + (r % 2 ? 0.5 : 0)) / counts[r]
      const p = pointAt(a)
      targets.push({ a: a % 1, x: p.x * s * SHRINK * f, y: -p.y * s * SHRINK * f })
    }
  })
  // Urutkan menurut sudut agar tiap foto bergerak kira-kira radial ke tengah
  targets.sort((p, q) => p.a - q.a)
  fillPts = targets

  // Tepi luar hati setelah merapat (dipakai untuk menjaga jarak video)
  outerRing = []
  for (let i = 0; i < heartPath.pts.length; i += 6) {
    outerRing.push({
      x: heartPath.pts[i].x * s * SHRINK,
      y: -heartPath.pts[i].y * s * SHRINK
    })
  }

  computePopLayout()
}

/* ---------- Posisi & ukuran 3 video pop-up ---------- */
// Jarak titik (px, py) ke persegi berpusat (cx, cy) berukuran w x h
function distToRect(px, py, cx, cy, w, h) {
  const dx = Math.max(Math.abs(px - cx) - w / 2, 0)
  const dy = Math.max(Math.abs(py - cy) - h / 2, 0)
  return Math.hypot(dx, dy)
}

function computePopLayout() {
  const vw = window.innerWidth
  const vh = window.innerHeight
  const pad = vw <= 480 ? 4 : 6
  popPad.value = pad

  // Ukuran kotak maksimum video (sudah dikali POP_SIZE)
  const boxW = clamp(vw * 0.2, 96, 190) * POP_SIZE
  const boxH = clamp(vh * 0.28, 130, 280) * POP_SIZE

  // Hati ikut berdenyut (scale ~1.03) dan foto sedikit berputar, jadi beri ruang ekstra
  const need = lastSize * 0.66 * 1.03 + POP_GAP
  const halfW = vw / 2 - POP_EDGE
  const halfH = vh / 2 - POP_EDGE
  const maxT = Math.max(vw, vh)

  const isClearOfHeart = (cx, cy, w, h) => {
    for (const q of outerRing) {
      if (distToRect(q.x, q.y, cx, cy, w, h) < need) return false
    }
    return true
  }

  const overlapsPlaced = (cx, cy, w, h, placed) =>
    placed.some(
      (p) => Math.abs(cx - p.x) < (w + p.w) / 2 + 12 && Math.abs(cy - p.y) < (h + p.h) / 2 + 12
    )

  // Cari posisi terdekat dari hati sepanjang beberapa arah
  const findSpot = (slot, w, h, placed) => {
    let fallback = null
    for (const deg of slot.angles) {
      const rad = (deg * Math.PI) / 180
      const ux = slot.sx * Math.cos(rad)
      const uy = slot.sy * Math.sin(rad)
      for (let t = 0; t <= maxT; t += 4) {
        const cx = ux * t
        const cy = uy * t
        if (Math.abs(cx) + w / 2 > halfW || Math.abs(cy) + h / 2 > halfH) break
        if (!fallback) fallback = { x: cx, y: cy }
        else if (deg === slot.angles[0]) fallback = { x: cx, y: cy }
        if (isClearOfHeart(cx, cy, w, h) && !overlapsPlaced(cx, cy, w, h, placed)) {
          return { x: cx, y: cy, ok: true }
        }
      }
    }
    return { x: fallback ? fallback.x : 0, y: fallback ? fallback.y : 0, ok: false }
  }

  // Muat rasio asli video ke dalam kotak (tanpa dipotong)
  const fit = (i, k) => {
    const r = popRatios.value[i] || 0.5625
    const bw = boxW * k
    const bh = boxH * k
    let w = bw
    let h = w / r
    if (h > bh) {
      h = bh
      w = h * r
    }
    return { w: Math.round(w), h: Math.round(h) }
  }

  // Kalau tidak muat di layar kecil, ukuran video dikecilkan bertahap sampai muat
  let result = null
  for (let k = 1; k >= 0.45; k -= 0.05) {
    const sizes = [0, 1, 2].map((i) => fit(i, k))
    const placed = []
    let allOk = true
    for (let i = 0; i < 3; i++) {
      const fw = sizes[i].w + pad * 2
      const fh = sizes[i].h + pad * 2
      const r = findSpot(POP_SLOTS[i], fw, fh, placed)
      placed.push({ x: r.x, y: r.y, w: fw, h: fh })
      if (!r.ok) allOk = false
    }
    result = { sizes, placed }
    if (allOk) break
  }

  popSizes.value = result.sizes
  popPos.value = result.placed.map((p) => ({ x: p.x, y: p.y }))
}

const stageStyle = computed(() => ({
  '--photo-size': `${photoSize.value}px`,
  '--pile-r': `${Math.round(photoSize.value * 1.15)}px`,
  '--zone-size': `${zoneSize.value}px`,
  '--orb-size': `${orbSize.value}px`,
  '--zone-y': `${zoneY.value}px`,
  '--pop-pad': `${popPad.value}px`
}))

// Posisi tiap video: titik pusat = tengah layar + offset (anchor dipusatkan oleh GSAP xPercent/yPercent)
function popStyle(i) {
  const p = popPos.value[i]
  return {
    left: `calc(50% + ${Math.round(p.x)}px)`,
    top: `calc(50% + ${Math.round(p.y)}px)`
  }
}

// Ukuran video mengikuti rasio aslinya
function videoStyle(i) {
  const s = popSizes.value[i]
  return { width: `${s.w}px`, height: `${s.h}px` }
}

// Setelah metadata termuat, pakai rasio asli video lalu hitung ulang posisi
function onVideoMeta(i, e) {
  const v = e.target
  if (v.videoWidth > 0 && v.videoHeight > 0) {
    const next = popRatios.value.slice()
    next[i] = v.videoWidth / v.videoHeight
    popRatios.value = next
    computePopLayout()
  }
}

/* ================= GSAP ================= */
function getEls() {
  return stageRef.value ? Array.from(stageRef.value.querySelectorAll('.photo-wrap')) : []
}

const prefersReduced = () => window.matchMedia('(prefers-reduced-motion: reduce)').matches

function startBeat(amount) {
  if (!groupRef.value || prefersReduced()) return
  beatTween?.kill()
  beatTween = gsap.to(groupRef.value, {
    scale: amount,
    duration: 0.9,
    repeat: -1,
    yoyo: true,
    ease: 'sine.inOut'
  })
}

function stopBeat() {
  beatTween?.kill()
  beatTween = null
  gsap.to(groupRef.value, { scale: 1, duration: 0.3, ease: 'power2.out' })
}

// State 1: 60 foto muncul (fade-in) dan bertumpuk berantakan di tengah
function playPile() {
  const els = getEls()
  const r = photoSize.value * 0.45
  gsap.set(els, {
    x: () => gsap.utils.random(-r, r),
    y: () => gsap.utils.random(-r, r),
    rotation: () => gsap.utils.random(-35, 35),
    scale: 0.6,
    opacity: 0
  })
  gsap.to(els, {
    opacity: 1,
    scale: 1,
    duration: 0.6,
    ease: 'back.out(1.7)',
    stagger: { each: 0.018, from: 'random' },
    onComplete: () => startBeat(1.03) // denyut halus sebagai isyarat "bisa diklik"
  })
}

// State 2: tumpukan meledak -> membentuk bingkai hati
function playExplode() {
  const els = getEls()
  gsap.killTweensOf(els)
  stopBeat()

  const tl = gsap.timeline({
    onComplete: () => {
      phase.value = 'heart'
      startBeat(1.02)
    }
  })

  // sedikit menekan ke dalam sebelum meledak
  tl.to(els, { scale: 0.85, duration: 0.18, ease: 'power2.in' })
  tl.fromTo(
    shockRef.value,
    { scale: 0, opacity: 0.9 },
    { scale: 14, opacity: 0, duration: 1.1, ease: 'power2.out' },
    '>'
  )
  tl.to(
    els,
    {
      x: (i) => outlinePts[i].x,
      y: (i) => outlinePts[i].y,
      rotation: () => gsap.utils.random(-16, 16),
      scale: 1,
      opacity: 1,
      duration: 1.5,
      ease: 'expo.out',
      stagger: { each: 0.01, from: 'random' }
    },
    '<'
  )
}

// State 5: foto merapat mengisi rongga. Setelah SELESAI -> 3 video pop-up muncul
function playShrink() {
  const els = getEls()
  gsap.killTweensOf(els)

  const tl = gsap.timeline({
    onComplete: () => {
      phase.value = 'filled'
      startBeat(1.03)
      revealPopupVideos()
    }
  })

  tl.to(els, {
    x: (i) => fillPts[i].x,
    y: (i) => fillPts[i].y,
    scale: 0.9,
    rotation: () => gsap.utils.random(-18, 18),
    duration: 2.2,
    ease: 'power3.inOut',
    stagger: { each: 0.015, from: 'random' }
  })
}

// Fade-in + scale up untuk 3 video pop-up (bergantian cepat)
function playVideosIn() {
  const wraps = stageRef.value ? Array.from(stageRef.value.querySelectorAll('.pop-video')) : []
  if (!wraps.length) return

  const tilts = [3, -3, -2]
  let done = 0
  let started = false

  const run = () => {
    if (started) return
    started = true
    gsap.to(wraps, {
      opacity: 1,
      scale: 1,
      rotation: (i) => tilts[i % tilts.length],
      duration: 0.8,
      ease: 'back.out(1.6)',
      stagger: 0.25
    })
  }

  // xPercent / yPercent memusatkan setiap video pada titik posisinya
  gsap.set(wraps, { xPercent: -50, yPercent: -50, opacity: 0, scale: 0.3, rotation: 0 })

  // Tunggu metadata video supaya frame sudah memakai rasio aslinya
  wraps.forEach((wrap) => {
    const vid = wrap.querySelector('video')
    if (!vid) {
      done++
      return
    }
    vid.muted = true
    const p = vid.play()
    if (p && typeof p.catch === 'function') p.catch(() => {})

    if (vid.readyState >= 1) {
      done++
    } else {
      vid.addEventListener(
        'loadedmetadata',
        () => {
          done++
          if (done >= wraps.length) run()
        },
        { once: true }
      )
    }
  })

  if (done >= wraps.length) run()
  // Cadangan: tetap tampilkan walau ada video yang gagal dimuat
  timers.push(setTimeout(run, 2500))
}

async function revealPopupVideos() {
  showVideos.value = true
  await nextTick()
  ctx?.add(playVideosIn)

  // State 6: Back muncul beberapa detik setelah video pop-up mulai muncul
  timers.push(setTimeout(() => (showBack.value = true), BACK_DELAY_MS))
}

/* ================= STATE 1 -> 2 ================= */
function explodeToHeart() {
  if (phase.value !== 'pile') return
  phase.value = 'exploding'
  showHint.value = false
  ctx.add(playExplode)
}

/* ================= STATE 4 -> 5 ================= */
function activateFinale() {
  if (phase.value !== 'heart') return
  phase.value = 'shrinking'
  orbVisible.value = false
  closeModal()
  stopBeat()
  ctx.add(playShrink)
}

/* ================= INTERAKSI ZONA RAHASIA ================= */
function onZoneEnter() {
  if (phase.value !== 'heart') return
  if (!orbVisible.value) lastShownAt = performance.now()
  orbVisible.value = true
}

function onZoneLeave(e) {
  if (e.pointerType === 'mouse') orbVisible.value = false
}

function onZoneFocus() {
  if (phase.value === 'heart') orbVisible.value = true
}

function onZoneBlur() {
  orbVisible.value = false
}

function onZonePointerDown(e) {
  // Di layar sentuh, ketukan pertama hanya memunculkan tombol
  hiddenOnDown =
    !orbVisible.value || (e.pointerType !== 'mouse' && performance.now() - lastShownAt < 400)
}

function onZoneClick(e) {
  if (phase.value !== 'heart') return
  const fromKeyboard = e.detail === 0
  if (!orbVisible.value) {
    orbVisible.value = true
    return
  }
  if (!fromKeyboard && hiddenOnDown) return
  activateFinale()
}

/* ================= MODAL ================= */
function openModal(photo) {
  if (phase.value !== 'heart') return
  selectedPhoto.value = photo.src
  isModalOpen.value = true
}

function closeModal() {
  isModalOpen.value = false
}

function onKeydown(e) {
  if (e.key === 'Escape' && isModalOpen.value) closeModal()
}

/* ================= LAIN-LAIN ================= */
function goBack() {
  emit('back-to-menu')
}

function onImgError(e) {
  e.target.classList.add('is-broken')
}

function onResize() {
  clearTimeout(resizeTimer)
  resizeTimer = setTimeout(() => {
    if (phase.value === 'exploding' || phase.value === 'shrinking') return
    computeLayout()
    if (phase.value === 'pile') return
    const els = getEls()
    const pts = phase.value === 'filled' ? fillPts : outlinePts
    gsap.killTweensOf(els)
    gsap.set(els, { x: (i) => pts[i].x, y: (i) => pts[i].y })
  }, 150)
}

/* ================= LIFECYCLE ================= */
onMounted(async () => {
  await nextTick()
  computeLayout()

  ctx = gsap.context(() => {}, stageRef.value)
  ctx.add(playPile)

  // Setelah 2 detik, munculkan teks petunjuk + panah
  timers.push(
    setTimeout(() => {
      if (phase.value === 'pile') showHint.value = true
    }, HINT_DELAY_MS)
  )

  window.addEventListener('resize', onResize)
  window.addEventListener('keydown', onKeydown)
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', onResize)
  window.removeEventListener('keydown', onKeydown)
  clearTimeout(resizeTimer)
  timers.forEach(clearTimeout)
  beatTween?.kill()
  ctx?.revert()
})
</script>

<style scoped>
.finale {
  /* Alias lokal dengan fallback, tetap mengikuti variabel root */
  --c-space: var(--bg-space, #0b0c10);
  --c-pink: var(--neon-pink, #ff1493);
  --c-paper: var(--bg-polaroid, #fafafa);
  --f-ui: var(--font-ui, 'Poppins', sans-serif);

  position: fixed;
  inset: 0;
  overflow: hidden;
  font-family: var(--f-ui);
  color: #fff;
  background:
    radial-gradient(ellipse at 50% 45%, rgba(255, 20, 147, 0.14), transparent 60%),
    repeating-linear-gradient(0deg, rgba(255, 20, 147, 0.05) 0 1px, transparent 1px 36px),
    repeating-linear-gradient(90deg, rgba(255, 20, 147, 0.05) 0 1px, transparent 1px 36px),
    var(--c-space);
}

/* ================= GELOMBANG KEJUT ================= */
.shock {
  position: absolute;
  z-index: 2;
  left: 50%;
  top: 50%;
  width: 100px;
  height: 100px;
  margin: -50px 0 0 -50px;
  border-radius: 50%;
  border: 2px solid var(--c-pink);
  box-shadow: 0 0 30px var(--c-pink), inset 0 0 30px rgba(255, 20, 147, 0.5);
  opacity: 0;
  pointer-events: none;
}

/* ================= FORMASI FOTO ================= */
.heart-group {
  position: absolute;
  inset: 0;
  z-index: 3;
  transform-origin: 50% 50%;
  pointer-events: none;
}

.photo-wrap {
  position: absolute;
  left: 50%;
  top: 50%;
  width: var(--photo-size);
  height: var(--photo-size);
  margin: calc(var(--photo-size) / -2) 0 0 calc(var(--photo-size) / -2);
  opacity: 0; /* dimunculkan oleh GSAP */
  will-change: transform;
}

.photo-wrap:hover,
.photo-wrap:focus-within {
  z-index: 30;
}

.photo {
  display: block;
  width: 100%;
  height: 100%;
  padding: 3px;
  border: none;
  border-radius: 2px;
  background: var(--c-paper);
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.55);
  cursor: default;
  pointer-events: none;
  transition: transform 0.25s cubic-bezier(0.2, 0.9, 0.3, 1.3), box-shadow 0.25s ease;
  -webkit-tap-highlight-color: transparent;
}

/* Foto hanya bisa di-hover / diklik saat hati sudah terbentuk (State 3) */
.finale.is-interactive .photo {
  pointer-events: auto;
  cursor: pointer;
}

.finale.is-interactive .photo:hover,
.finale.is-interactive .photo:focus-visible {
  transform: scale(1.35);
  box-shadow: 0 0 16px var(--c-pink), 0 8px 20px rgba(0, 0, 0, 0.6);
  outline: none;
}

.photo img {
  display: block;
  width: 100%;
  height: 100%;
  object-fit: cover;
  background: linear-gradient(135deg, #ffd1e6, #ff8fc4);
  user-select: none;
  -webkit-user-drag: none;
}

.photo img.is-broken {
  visibility: hidden;
}

/* ================= STATE 5: 3 VIDEO POP-UP ================= */
/* Tetap berada di belakang hati (z-index 2 < 3), tetapi posisinya sudah diberi jarak dari hati.
   left/top diisi lewat inline style, titik pusat diatur GSAP lewat xPercent/yPercent.
   Ukuran video (width/height) diisi lewat inline style mengikuti rasio aslinya. */
.pop-video {
  position: absolute;
  z-index: 2;
  padding: var(--pop-pad, 6px);
  background: var(--c-paper);
  border-radius: 8px;
  box-shadow: 0 0 18px rgba(255, 20, 147, 0.55), 0 10px 26px rgba(0, 0, 0, 0.55);
  opacity: 0; /* dimunculkan oleh GSAP */
  pointer-events: none;
  will-change: transform, opacity;
}

.pop-video video {
  display: block;
  border-radius: 4px;
  background: #111;
}

/* ================= STATE 1: TUMPUKAN & PETUNJUK ================= */
.pile-hit {
  position: absolute;
  z-index: 6;
  left: 50%;
  top: 50%;
  width: calc(var(--pile-r) * 2);
  height: calc(var(--pile-r) * 2);
  margin: calc(var(--pile-r) * -1) 0 0 calc(var(--pile-r) * -1);
  padding: 0;
  border: 2px dashed rgba(255, 20, 147, 0.75);
  border-radius: 50%;
  background: transparent;
  cursor: pointer;
  outline: none;
  animation: ringPulse 1.6s ease-in-out infinite;
  -webkit-tap-highlight-color: transparent;
}

.pile-hit:focus-visible {
  box-shadow: 0 0 0 3px #fff, 0 0 20px var(--c-pink);
}

.pile-hint {
  position: absolute;
  z-index: 7;
  left: 50%;
  bottom: calc(50% + var(--pile-r) + 10px);
  transform: translateX(-50%);
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  width: max-content;
  max-width: 92vw;
  text-align: center;
  pointer-events: none;
}

.hint-text {
  margin: 0;
  font-size: clamp(1.25rem, 4.4vw, 2rem);
  font-weight: 600;
  letter-spacing: 0.5px;
  color: #fff;
  text-shadow: 0 0 10px var(--c-pink), 0 0 26px rgba(255, 20, 147, 0.8);
  animation: textPulse 1.3s ease-in-out infinite;
}

.arrows {
  display: flex;
  flex-direction: column;
  align-items: center;
  line-height: 0;
}

.arrow {
  width: clamp(30px, 7vw, 42px);
  height: clamp(30px, 7vw, 42px);
  margin-top: -12px;
  fill: none;
  stroke: var(--c-pink);
  stroke-width: 3;
  stroke-linecap: round;
  stroke-linejoin: round;
  filter: drop-shadow(0 0 6px var(--c-pink));
  animation: arrowDrop 1.2s ease-in-out infinite;
}

/* ================= STATE 4: TOMBOL RAHASIA ================= */
.secret-zone {
  position: absolute;
  z-index: 5;
  left: 50%;
  top: 50%;
  width: var(--zone-size);
  height: var(--zone-size);
  margin: calc(var(--zone-size) / -2) 0 0 calc(var(--zone-size) / -2);
  transform: translateY(var(--zone-y));
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  border: none;
  border-radius: 50%;
  background: transparent; /* tak terlihat sampai di-hover */
  outline: none;
  pointer-events: none;
  -webkit-tap-highlight-color: transparent;
}

.secret-zone.is-ready {
  pointer-events: auto;
}

.secret-zone.is-active {
  cursor: pointer;
}

.orb {
  width: var(--orb-size);
  height: var(--orb-size);
  border-radius: 50%;
  background: radial-gradient(circle at 35% 30%, #ffffff 0%, var(--c-pink) 45%, rgba(255, 20, 147, 0.25) 100%);
  box-shadow: 0 0 24px var(--c-pink), 0 0 60px rgba(255, 20, 147, 0.6);
  opacity: 0;
  transform: scale(0.3);
  transition: opacity 0.4s ease, transform 0.5s cubic-bezier(0.2, 0.9, 0.3, 1.4);
  pointer-events: none;
}

.secret-zone.is-active .orb,
.secret-zone:focus-visible .orb {
  opacity: 1;
  transform: scale(1);
  animation: orbGlow 1.8s ease-in-out infinite;
}

/* Saat diklik: orb membesar lalu menghilang */
.secret-zone.is-burst .orb {
  opacity: 0;
  transform: scale(2.4);
  transition: opacity 0.6s ease, transform 0.6s ease;
  animation: none;
}

/* ================= PETUNJUK BAWAH ================= */
.hint-bottom {
  position: absolute;
  z-index: 8;
  left: 50%;
  bottom: max(22px, env(safe-area-inset-bottom));
  transform: translateX(-50%);
  margin: 0;
  width: max-content;
  max-width: 90vw;
  text-align: center;
  font-size: 0.85rem;
  color: rgba(255, 255, 255, 0.75);
  pointer-events: none;
  animation: hintPulse 2.4s ease-in-out infinite;
}

/* ================= TOMBOL BACK ================= */
.back-btn {
  position: fixed;
  z-index: 120;
  top: max(16px, env(safe-area-inset-top));
  left: 16px;
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 10px 18px 10px 12px;
  border: 1.5px solid var(--c-pink);
  border-radius: 999px;
  background: rgba(11, 12, 16, 0.6);
  backdrop-filter: blur(6px);
  -webkit-backdrop-filter: blur(6px);
  color: #fff;
  font-family: var(--f-ui);
  font-size: 0.9rem;
  font-weight: 500;
  cursor: pointer;
  box-shadow: 0 0 14px rgba(255, 20, 147, 0.5);
  transition: transform 0.2s ease, box-shadow 0.2s ease;
  -webkit-tap-highlight-color: transparent;
}

.back-btn svg {
  width: 20px;
  height: 20px;
  fill: none;
  stroke: currentColor;
  stroke-width: 2.4;
  stroke-linecap: round;
  stroke-linejoin: round;
}

.back-btn:hover {
  transform: translateX(-2px);
  box-shadow: 0 0 22px rgba(255, 20, 147, 0.8);
}

/* ================= MODAL POLAROID ================= */
.modal {
  position: fixed;
  inset: 0;
  z-index: 100;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
  background: rgba(5, 5, 10, 0.78);
  backdrop-filter: blur(5px);
  -webkit-backdrop-filter: blur(5px);
}

.modal-polaroid {
  position: relative;
  width: min(88vw, 440px);
  padding: 14px 14px 20px;
  box-sizing: border-box;
  background: var(--c-paper);
  border-radius: 4px;
  box-shadow: 0 0 30px rgba(255, 20, 147, 0.55), 0 24px 60px rgba(0, 0, 0, 0.6);
  text-align: center;
  transform: rotate(-2deg);
}

.modal-photo {
  width: 100%;
  aspect-ratio: 1 / 1;
  background: linear-gradient(135deg, #ffd1e6, #ff8fc4);
  overflow: hidden;
}

.modal-photo img {
  display: block;
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.modal-photo img.is-broken {
  visibility: hidden;
}

.modal-caption {
  margin: 16px 0 14px;
  font-family: var(--f-ui);
  font-size: 1rem;
  color: var(--text-dark, #202020);
}

.close-btn {
  padding: 10px 28px;
  border: none;
  border-radius: 999px;
  background: var(--c-pink);
  color: #fff;
  font-family: var(--f-ui);
  font-size: 0.9rem;
  font-weight: 600;
  cursor: pointer;
  box-shadow: 0 0 16px rgba(255, 20, 147, 0.6);
  transition: transform 0.2s ease;
}

.close-btn:hover {
  transform: scale(1.06);
}

.close-btn:active {
  transform: scale(0.97);
}

/* ================= TRANSISI VUE ================= */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.6s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.3s ease;
}

.modal-enter-active .modal-polaroid,
.modal-leave-active .modal-polaroid {
  transition: transform 0.4s cubic-bezier(0.2, 0.9, 0.3, 1.2), opacity 0.3s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}

.modal-enter-from .modal-polaroid,
.modal-leave-to .modal-polaroid {
  opacity: 0;
  transform: rotate(-8deg) scale(0.7) translateY(30px);
}

/* ================= KEYFRAMES ================= */
@keyframes textPulse {
  0%,
  100% {
    opacity: 0.55;
    transform: scale(0.97);
  }
  50% {
    opacity: 1;
    transform: scale(1.03);
  }
}

@keyframes arrowDrop {
  0%,
  100% {
    opacity: 0.25;
    transform: translateY(-6px);
  }
  50% {
    opacity: 1;
    transform: translateY(6px);
  }
}

@keyframes ringPulse {
  0%,
  100% {
    opacity: 0.35;
    box-shadow: 0 0 0 rgba(255, 20, 147, 0);
  }
  50% {
    opacity: 1;
    box-shadow: 0 0 22px rgba(255, 20, 147, 0.7);
  }
}

@keyframes orbGlow {
  0%,
  100% {
    box-shadow: 0 0 18px var(--c-pink), 0 0 40px rgba(255, 20, 147, 0.45);
  }
  50% {
    box-shadow: 0 0 34px var(--c-pink), 0 0 80px rgba(255, 20, 147, 0.85);
  }
}

@keyframes hintPulse {
  0%,
  100% {
    opacity: 0.45;
  }
  50% {
    opacity: 0.9;
  }
}

/* ================= RESPONSIF ================= */
@media (max-width: 480px) {
  .hint-bottom {
    font-size: 0.75rem;
  }

  .back-btn {
    padding: 8px 14px 8px 10px;
    font-size: 0.82rem;
  }

  .pop-video {
    border-radius: 6px;
  }
}

@media (prefers-reduced-motion: reduce) {
  .pile-hit,
  .hint-text,
  .arrow,
  .hint-bottom,
  .secret-zone.is-active .orb {
    animation: none;
  }
}
</style>