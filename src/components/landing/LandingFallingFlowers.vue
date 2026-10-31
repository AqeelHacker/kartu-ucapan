<!-- LandingFallingFlowers.vue : Bunga melayang di sisi kiri & kanan layar (5 bunga bergantian) -->
<template>
  <div class="falling-layer" aria-hidden="true">
    <div
      v-for="slot in slots"
      :key="slot.id"
      class="fall"
      :style="{
        left: slot.x + 'vw',
        fontSize: slot.size + 'rem',
        '--dur': slot.dur + 's',
        '--delay': slot.delay + 's',
        '--drift': slot.drift + 'px'
      }"
      @animationend.self="respawn(slot.index)"
    >
      <span
        class="sway"
        :style="{
          '--sway': slot.sway + 'px',
          '--spin': slot.spin + 'deg',
          '--sway-dur': slot.swayDur + 's'
        }"
      >
        {{ slot.char }}
      </span>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'

/* ================= EDIT DI SINI ================= */
const SLOT_COUNT = 5 // jumlah bunga yang jatuh bersamaan
const FLOWER_TYPES = ['🌸', '🌹', '🌷', '🌺', '🌻', '🌼', '🪷', '💮', '🏵️', '💐'] // 10 jenis bunga
const DURATION_MIN = 9 // detik, kecepatan jatuh paling cepat
const DURATION_MAX = 14 // detik, kecepatan jatuh paling lambat
const SIZE_MIN = 1.5 // rem
const SIZE_MAX = 2.6 // rem
const STAGGER_S = 2.4 // jeda antar bunga di awal (supaya gantian)
/* ================================================= */

let idCounter = 0
let sideToggle = 0
let lastChar = ''

const rand = (min, max) => min + Math.random() * (max - min)
const r1 = (n) => Math.round(n * 10) / 10

function pickChar() {
  let c
  do {
    c = FLOWER_TYPES[Math.floor(Math.random() * FLOWER_TYPES.length)]
  } while (c === lastChar && FLOWER_TYPES.length > 1)
  lastChar = c
  return c
}

function makeSlot(index, initial) {
  // Selang-seling kiri / kanan. Zona dipilih agar di laptop jatuh di area kosong samping konten.
  sideToggle = 1 - sideToggle
  const x = sideToggle === 0 ? rand(2, 17) : rand(83, 96)

  return {
    index,
    id: idCounter++,
    char: pickChar(),
    x: r1(x),
    size: r1(rand(SIZE_MIN, SIZE_MAX)),
    dur: r1(rand(DURATION_MIN, DURATION_MAX)),
    // awal: bergiliran muncul; sesudahnya: jeda acak singkat
    delay: r1(initial ? index * STAGGER_S + rand(0, 1) : rand(0.4, 2.5)),
    drift: Math.round(rand(-40, 40)),
    sway: Math.round(rand(14, 34)),
    spin: Math.round(rand(14, 40)),
    swayDur: r1(rand(2.8, 4.6))
  }
}

const slots = ref(Array.from({ length: SLOT_COUNT }, (_, i) => makeSlot(i, true)))

// Bunga selesai jatuh -> slot diganti bunga baru (key berubah, animasi mulai ulang)
function respawn(index) {
  slots.value[index] = makeSlot(index, false)
}
</script>

<style scoped>
.falling-layer {
  position: fixed;
  inset: 0;
  z-index: 0;
  overflow: hidden;
  pointer-events: none;
}

.fall {
  position: absolute;
  top: -70px;
  line-height: 1;
  opacity: 0;
  will-change: transform, opacity;
  animation: fall var(--dur, 11s) linear var(--delay, 0s) 1 both;
}

.sway {
  display: block;
  filter: drop-shadow(0 0 10px rgba(255, 105, 170, 0.45));
  animation: sway var(--sway-dur, 3.5s) ease-in-out infinite alternate;
}

@keyframes fall {
  0% {
    transform: translate3d(0, 0, 0);
    opacity: 0;
  }
  8% {
    opacity: 0.9;
  }
  90% {
    opacity: 0.9;
  }
  100% {
    transform: translate3d(var(--drift, 0px), calc(100vh + 150px), 0);
    opacity: 0;
  }
}

@keyframes sway {
  from {
    transform: translateX(calc(var(--sway, 20px) * -1)) rotate(calc(var(--spin, 25deg) * -1));
  }
  to {
    transform: translateX(var(--sway, 20px)) rotate(var(--spin, 25deg));
  }
}

/* Di HP, sisi layar sempit, jadi bunga dibuat lebih samar dan berada di belakang konten */
@media (max-width: 700px) {
  .falling-layer {
    opacity: 0.5;
  }
}

@media (prefers-reduced-motion: reduce) {
  .sway {
    animation: none;
  }
}
</style>