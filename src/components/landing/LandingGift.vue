<!-- LandingGift.vue : Gerbang menuju Fase 1 + tombol Exit -->
<template>
  <section ref="rootRef" class="gift-section">
    <!-- Tombol Exit (pojok kanan atas): reset ke layar sandi + matikan musik -->
    <button class="exit-btn" type="button" aria-label="Keluar" @click="onExit">
      <svg viewBox="0 0 24 24" aria-hidden="true">
        <path d="M6 6l12 12M18 6L6 18" />
      </svg>
      <span>Exit</span>
    </button>

    <div class="gift-block" data-reveal>
      <p class="eyebrow">— One last thing —</p>
      <h2 class="title">
        {{ isOpen ? 'Surprise! 🎉' : isOpening ? 'Opening your gift…' : 'A Little Surprise' }}
      </h2>

      <div class="gift-stage">
        <button
          class="gift"
          :class="{ 'is-shaking': isShaking, 'is-opening': isOpening, 'is-open': isOpen }"
          type="button"
          aria-label="Buka kado"
          :disabled="isOpening"
          @click="openGift"
        >
          <span class="rays" aria-hidden="true"></span>
          <span class="halo" aria-hidden="true"></span>
          <span class="flash" aria-hidden="true"></span>

          <svg class="gift-svg" viewBox="0 0 200 200" aria-hidden="true">
            <defs>
              <linearGradient id="gf-body" x1="0" y1="0" x2="0" y2="1">
                <stop offset="0" stop-color="#f9a3c3" />
                <stop offset="1" stop-color="#e45f92" />
              </linearGradient>
              <linearGradient id="gf-lid" x1="0" y1="0" x2="0" y2="1">
                <stop offset="0" stop-color="#ffb4cf" />
                <stop offset="1" stop-color="#ee74a2" />
              </linearGradient>
              <linearGradient id="gf-ribbon" x1="0" y1="0" x2="0" y2="1">
                <stop offset="0" stop-color="#ffe066" />
                <stop offset="1" stop-color="#ffbb1f" />
              </linearGradient>
            </defs>

            <!-- BADAN -->
            <rect x="38" y="100" width="124" height="86" rx="8" fill="url(#gf-body)" />
            <!-- pita vertikal badan -->
            <rect x="90" y="100" width="20" height="86" fill="url(#gf-ribbon)" />
            <!-- kilau -->
            <rect x="46" y="108" width="8" height="64" rx="4" fill="#ffffff" opacity="0.18" />
            <!-- mulut kotak (terlihat setelah tutup terbuka) -->
            <ellipse class="mouth" cx="100" cy="102" rx="58" ry="8" fill="#6b1840" />
            <ellipse class="mouth-glow" cx="100" cy="102" rx="46" ry="4.5" fill="#ffd9ea" />

            <!-- TUTUP (ikut terangkat saat dibuka) -->
            <g class="lid">
              <rect x="28" y="72" width="144" height="34" rx="8" fill="url(#gf-lid)" />
              <rect x="90" y="72" width="20" height="34" fill="url(#gf-ribbon)" />
              <rect x="28" y="84" width="144" height="10" fill="url(#gf-ribbon)" opacity="0.95" />
              <!-- simpul -->
              <path
                d="M100 72 C84 44 54 46 62 64 C68 76 90 74 100 72 Z"
                fill="url(#gf-ribbon)"
              />
              <path
                d="M100 72 C116 44 146 46 138 64 C132 76 110 74 100 72 Z"
                fill="url(#gf-ribbon)"
              />
              <circle cx="100" cy="70" r="9" fill="#ffcc33" />
            </g>
          </svg>
        </button>

        <!-- Bunga & percikan yang meledak keluar dari kado -->
        <div class="burst-layer" aria-hidden="true">
          <span
            v-for="p in burst"
            :key="p.id"
            class="burst"
            :style="{
              '--dx': p.dx + 'px',
              '--dy': p.dy + 'px',
              '--rot': p.rot + 'deg',
              fontSize: p.size + 'rem',
              animationDelay: p.delay + 's',
              animationDuration: p.dur + 's'
            }"
          >
            {{ p.char }}
          </span>
        </div>
      </div>

      <p class="tap-hint" :class="{ gone: isOpening }">Sentuh untuk kejutan terakhir</p>
    </div>

    <!-- Layar menggelap perlahan sebelum pindah ke Fase 1 -->
    <Teleport to="body">
      <div class="fade-black" :class="{ on: isFading }" aria-hidden="true"></div>
    </Teleport>
  </section>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'

/*
  Emit:
  - 'start-surprise' : pindah ke Fase 1 (App.vue: currentPhase = 1)
  - 'exit'           : reset ke layar sandi + matikan musik (ditangani LandingView / App.vue)
*/
const emit = defineEmits(['start-surprise', 'exit'])

/* ================= EDIT DI SINI ================= */
const SHAKE_MS = 700 // lama kado bergetar sebelum terbuka
const OPEN_AT_MS = 700 // tutup kado terbuka + bunga meledak (sesudah getar)
const DARKEN_AT_MS = 2400 // layar mulai menggelap
const GO_AT_MS = 3900 // emit pindah ke Fase 1 (layar sudah hitam penuh)

const BURST_COUNT = 20 // jumlah bunga/percikan yang meledak
const BURST_CHARS = ['🌸', '🌹', '🌷', '🌺', '🌻', '🌼', '🪷', '💮', '🏵️', '💐', '✨', '💖']

// Efek suara saat kado terbuka (kosongkan '' kalau tidak mau). Contoh: '/audio/pop.mp3'
const OPEN_SFX_SRC = ''
const OPEN_SFX_VOLUME = 0.7
/* ================================================= */

const rootRef = ref(null)
const isOpening = ref(false) // dari klik sampai pindah fase
const isShaking = ref(false)
const isOpen = ref(false) // tutup sudah terbuka
const isFading = ref(false)
const burst = ref([])
const timers = []

function makeBurst() {
  const w = typeof window !== 'undefined' ? window.innerWidth : 400
  const h = typeof window !== 'undefined' ? window.innerHeight : 700
  const spreadX = Math.min(w * 0.44, 260) // sebaran kiri-kanan
  const riseMax = Math.min(h * 0.45, 320) // tinggi lontaran maksimum

  return Array.from({ length: BURST_COUNT }, (_, i) => ({
    id: i,
    char: BURST_CHARS[Math.floor(Math.random() * BURST_CHARS.length)],
    dx: Math.round((Math.random() * 2 - 1) * spreadX),
    dy: -Math.round(70 + Math.random() * Math.max(riseMax - 70, 40)),
    rot: Math.round((Math.random() * 2 - 1) * 360),
    size: (1.1 + Math.random() * 1.3).toFixed(2),
    delay: (Math.random() * 0.25).toFixed(2),
    dur: (1.7 + Math.random() * 0.8).toFixed(2)
  }))
}

function playOpenSfx() {
  if (!OPEN_SFX_SRC) return
  try {
    const s = new Audio(OPEN_SFX_SRC)
    s.volume = OPEN_SFX_VOLUME
    s.play().catch(() => {})
  } catch (e) {
    /* abaikan kalau audio tidak tersedia */
  }
}

function popOpen() {
  isOpen.value = true
  burst.value = makeBurst()
  playOpenSfx()
}

function openGift() {
  if (isOpening.value) return
  isOpening.value = true
  isShaking.value = true

  timers.push(setTimeout(() => (isShaking.value = false), SHAKE_MS))
  timers.push(setTimeout(popOpen, OPEN_AT_MS))
  timers.push(setTimeout(() => (isFading.value = true), DARKEN_AT_MS))
  timers.push(setTimeout(() => emit('start-surprise'), GO_AT_MS))
}

function onExit() {
  timers.forEach(clearTimeout)
  isOpening.value = false
  isShaking.value = false
  isOpen.value = false
  isFading.value = false
  burst.value = []
  emit('exit')
}

// Muncul perlahan saat di-scroll ke area ini
let io = null
onMounted(() => {
  const els = rootRef.value ? rootRef.value.querySelectorAll('[data-reveal]') : []
  if (!('IntersectionObserver' in window)) {
    els.forEach((el) => el.classList.add('is-visible'))
    return
  }
  io = new IntersectionObserver(
    (entries) => {
      entries.forEach((entry) => {
        if (entry.isIntersecting) {
          entry.target.classList.add('is-visible')
          io.unobserve(entry.target)
        }
      })
    },
    { threshold: 0.2 }
  )
  els.forEach((el) => io.observe(el))
})

onBeforeUnmount(() => {
  timers.forEach(clearTimeout)
  io?.disconnect()
})
</script>

<style scoped>
.gift-section {
  --ff-serif: 'Cormorant Garamond', 'Playfair Display', Georgia, serif;
  --pink: var(--neon-pink, #ff1493);
  --rose: #ffa6c9;

  position: relative;
  min-height: 100svh;
  padding: 70px 20px 90px;
  box-sizing: border-box;
  display: flex;
  align-items: center;
  justify-content: center;
  text-align: center;
  font-family: var(--ff-serif);
  color: #f8dce8;
}

[data-reveal] {
  opacity: 0;
  transform: translateY(26px);
  transition: opacity 1s ease, transform 1s cubic-bezier(0.2, 0.8, 0.25, 1);
}

[data-reveal].is-visible {
  opacity: 1;
  transform: translateY(0);
}

/* "— One last thing —" (disamakan dengan eyebrow di Bouquet) */
.eyebrow {
  margin: 0 0 12px;
  font-size: 0.95rem;
  font-weight: 500;
  letter-spacing: 3px;
  text-transform: uppercase;
  color: rgba(255, 194, 220, 0.95);
  text-shadow: 0 0 12px rgba(255, 105, 170, 0.35);
}

.title {
  margin: 0 0 18px;
  min-height: 2.8rem;
  font-size: clamp(1.9rem, 8vw, 2.5rem);
  font-weight: 400;
  color: #fbe7ef;
  text-shadow: 0 0 20px rgba(255, 105, 170, 0.3);
}

/* ================= KADO ================= */
.gift-stage {
  position: relative;
  width: min(62vw, 240px);
  aspect-ratio: 1;
  margin: 0 auto;
}

.gift {
  position: relative;
  width: 100%;
  height: 100%;
  margin: 0;
  padding: 0;
  border: none;
  background: transparent;
  cursor: pointer;
  -webkit-tap-highlight-color: transparent;
}

.gift:disabled {
  cursor: default;
}

.gift-svg {
  position: relative;
  z-index: 2;
  width: 100%;
  height: 100%;
  overflow: visible;
  filter: drop-shadow(0 10px 22px rgba(0, 0, 0, 0.45)) drop-shadow(0 0 18px rgba(255, 105, 170, 0.5));
  animation: giftFloat 3s ease-in-out infinite;
}

.halo {
  position: absolute;
  z-index: 1;
  inset: -22%;
  border-radius: 50%;
  background: radial-gradient(circle, rgba(255, 105, 170, 0.5) 0%, rgba(255, 20, 147, 0.18) 42%, transparent 70%);
  animation: haloPulse 2.2s ease-in-out infinite;
}

.rays {
  position: absolute;
  z-index: 0;
  inset: -90%;
  border-radius: 50%;
  background: repeating-conic-gradient(
    rgba(255, 166, 201, 0.2) 0deg 4deg,
    transparent 4deg 22.5deg
  );
  -webkit-mask-image: radial-gradient(circle, #000 12%, transparent 62%);
  mask-image: radial-gradient(circle, #000 12%, transparent 62%);
  opacity: 0;
  transition: opacity 0.6s ease;
  animation: raysSpin 14s linear infinite;
  pointer-events: none;
}

.gift:hover .rays,
.gift.is-opening .rays {
  opacity: 1;
}

/* Fase 1: bergetar */
.gift.is-shaking .gift-svg {
  animation: shake 0.16s linear infinite;
}

.gift.is-opening .halo {
  animation: haloBurst 1.7s ease-in forwards;
}

.gift.is-opening .gift-svg {
  filter: drop-shadow(0 0 30px #fff) drop-shadow(0 0 40px var(--pink));
}

/* Kilatan cahaya saat tutup terbuka */
.flash {
  position: absolute;
  z-index: 3;
  inset: -40%;
  border-radius: 50%;
  background: radial-gradient(circle, rgba(255, 255, 255, 0.95) 0%, rgba(255, 170, 210, 0.55) 35%, transparent 68%);
  opacity: 0;
  pointer-events: none;
}

/* Fase 2: tutup terbuka */
.lid {
  transform-box: fill-box;
  transform-origin: 0% 100%;
}

.mouth,
.mouth-glow {
  opacity: 0;
  transition: opacity 0.25s ease;
}

.gift.is-open .lid {
  animation: lidOpen 0.75s cubic-bezier(0.2, 1.15, 0.3, 1) forwards;
}

.gift.is-open .mouth,
.gift.is-open .mouth-glow {
  opacity: 1;
}

.gift.is-open .mouth-glow {
  animation: mouthPulse 0.5s ease-in-out infinite alternate;
}

.gift.is-open .flash {
  animation: flash 0.9s ease-out forwards;
}

.gift.is-open .gift-svg {
  animation: giftPop 0.45s ease-out 1, giftFloat 3s ease-in-out 0.45s infinite;
}

/* ================= BUNGA MELEDAK ================= */
.burst-layer {
  position: absolute;
  inset: 0;
  z-index: 4;
  pointer-events: none;
}

.burst {
  position: absolute;
  left: 50%;
  top: 44%; /* sekitar mulut kotak */
  line-height: 1;
  opacity: 0;
  filter: drop-shadow(0 0 8px rgba(255, 105, 170, 0.5));
  animation-name: burst;
  animation-timing-function: cubic-bezier(0.15, 0.7, 0.3, 1);
  animation-fill-mode: both;
}

.tap-hint {
  margin: 22px 0 0;
  font-size: 1rem;
  font-style: italic;
  letter-spacing: 0.5px;
  color: rgba(248, 220, 232, 0.7);
  animation: hintPulse 1.8s ease-in-out infinite;
  transition: opacity 0.4s ease;
}

.tap-hint.gone {
  opacity: 0;
}

/* ================= EXIT ================= */
.exit-btn {
  position: fixed;
  z-index: 40;
  top: max(16px, env(safe-area-inset-top));
  right: 16px;
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 8px 14px 8px 10px;
  border-radius: 999px;
  border: 1px solid rgba(255, 166, 201, 0.35);
  background: rgba(40, 8, 28, 0.55);
  backdrop-filter: blur(6px);
  -webkit-backdrop-filter: blur(6px);
  color: #f8dce8;
  font-family: var(--ff-serif);
  font-size: 0.95rem;
  letter-spacing: 0.5px;
  cursor: pointer;
  box-shadow: 0 0 14px rgba(255, 20, 147, 0.25);
  transition: box-shadow 0.2s ease, transform 0.2s ease;
  -webkit-tap-highlight-color: transparent;
}

.exit-btn svg {
  width: 16px;
  height: 16px;
  fill: none;
  stroke: currentColor;
  stroke-width: 2.2;
  stroke-linecap: round;
}

.exit-btn:hover {
  box-shadow: 0 0 20px rgba(255, 20, 147, 0.6);
  transform: scale(1.04);
}

/* ================= LAYAR HITAM (jembatan sinematik) ================= */
.fade-black {
  position: fixed;
  inset: 0;
  z-index: 9999;
  background: #000;
  opacity: 0;
  pointer-events: none;
  transition: opacity 1.4s ease-in;
}

.fade-black.on {
  opacity: 1;
  pointer-events: auto; /* blok klik selama transisi */
}

/* ================= KEYFRAMES ================= */
@keyframes giftFloat {
  0%,
  100% {
    transform: translateY(0) rotate(-1.5deg);
  }
  50% {
    transform: translateY(-9px) rotate(1.5deg);
  }
}

@keyframes giftPop {
  0% {
    transform: scale(1);
  }
  35% {
    transform: scale(1.12, 0.92);
  }
  70% {
    transform: scale(0.97, 1.05);
  }
  100% {
    transform: scale(1);
  }
}

@keyframes lidOpen {
  0% {
    transform: translate(0, 0) rotate(0deg);
  }
  100% {
    transform: translate(-30px, -64px) rotate(-30deg);
  }
}

@keyframes mouthPulse {
  from {
    opacity: 0.6;
  }
  to {
    opacity: 1;
  }
}

@keyframes flash {
  0% {
    opacity: 0;
    transform: scale(0.4);
  }
  25% {
    opacity: 0.9;
  }
  100% {
    opacity: 0;
    transform: scale(1.5);
  }
}

@keyframes burst {
  0% {
    opacity: 0;
    transform: translate(-50%, -50%) scale(0.2) rotate(0deg);
  }
  12% {
    opacity: 1;
  }
  55% {
    opacity: 1;
    transform: translate(calc(-50% + var(--dx)), calc(-50% + var(--dy))) scale(1) rotate(var(--rot));
  }
  100% {
    opacity: 0;
    transform: translate(calc(-50% + var(--dx) * 1.15), calc(-50% + var(--dy) + 150px)) scale(0.9)
      rotate(calc(var(--rot) * 1.6));
  }
}

@keyframes haloPulse {
  0%,
  100% {
    opacity: 0.6;
    transform: scale(0.92);
  }
  50% {
    opacity: 1;
    transform: scale(1.08);
  }
}

@keyframes haloBurst {
  0% {
    opacity: 0.8;
    transform: scale(1);
  }
  100% {
    opacity: 1;
    transform: scale(3.2);
  }
}

@keyframes raysSpin {
  to {
    transform: rotate(360deg);
  }
}

@keyframes shake {
  0% {
    transform: translate(0, 0) rotate(0deg);
  }
  20% {
    transform: translate(-6px, 2px) rotate(-5deg);
  }
  40% {
    transform: translate(5px, -3px) rotate(4deg);
  }
  60% {
    transform: translate(-5px, -2px) rotate(-4deg);
  }
  80% {
    transform: translate(6px, 3px) rotate(5deg);
  }
  100% {
    transform: translate(0, 0) rotate(0deg);
  }
}

@keyframes hintPulse {
  0%,
  100% {
    opacity: 0.45;
  }
  50% {
    opacity: 1;
  }
}

@media (prefers-reduced-motion: reduce) {
  .gift-svg,
  .halo,
  .rays,
  .tap-hint {
    animation: none;
  }

  [data-reveal] {
    transform: none;
  }
}
</style>