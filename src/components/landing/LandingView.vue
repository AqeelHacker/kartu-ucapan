<!-- LandingView.vue : Pembungkus Fase 0 (Halaman Awal) -->
<template>
  <Transition name="landing" mode="out-in">
    <!-- Langkah 0: layar loading singkat (hanya saat belum terbuka) -->
    <div v-if="showLoading" key="loading" class="loading" role="status">
      <svg class="loading-flower" viewBox="0 0 64 64" aria-hidden="true">
        <defs>
          <radialGradient id="lv-petal" cx="50%" cy="30%" r="80%">
            <stop offset="0" stop-color="#ffc2dc" />
            <stop offset="1" stop-color="#f06fa4" />
          </radialGradient>
        </defs>
        <ellipse
          v-for="i in 5"
          :key="i"
          cx="32"
          cy="17"
          rx="9.5"
          ry="14.5"
          fill="url(#lv-petal)"
          :transform="`rotate(${(i - 1) * 72} 32 32)`"
        />
        <circle cx="32" cy="32" r="6.5" fill="#ffe3ef" />
      </svg>
      <p class="loading-text">Preparing something special for you…</p>
    </div>

    <!-- Langkah 1: layar kunci -->
    <LandingPasscode
      v-else-if="!isUnlocked"
      key="passcode"
      @play-music="emit('play-music')"
      @unlocked="onUnlocked"
    />

    <!-- Langkah 2: buket + surat + kado (satu halaman yang bisa di-scroll) -->
    <div v-else key="content" class="landing-scroll">
      <!-- Bunga melayang di belakang konten (terlihat di sisi kiri/kanan pada laptop) -->
      <LandingFallingFlowers />

      <div class="landing-content">
        <LandingBouquet />
        <LandingGift @start-surprise="emit('start-surprise')" @exit="onExit" />
      </div>
    </div>
  </Transition>
</template>

<script setup>
import { ref, onBeforeUnmount } from 'vue'
import LandingPasscode from './LandingPasscode.vue'
import LandingBouquet from './LandingBouquet.vue'
import LandingGift from './LandingGift.vue'
import LandingFallingFlowers from './LandingFallingFlowers.vue'

/*
  isUnlocked disimpan di App.vue (prop `unlocked`, v-model:unlocked) karena LandingView
  di-unmount saat currentPhase berpindah. Jadi ketika tombol Back dari Fase 4 mengembalikan
  ke Fase 0, state ini tetap true dan Buket langsung tampil tanpa sandi.
*/
const props = defineProps({
  unlocked: { type: Boolean, default: false }
})

const emit = defineEmits(['update:unlocked', 'play-music', 'stop-music', 'start-surprise'])

const LOADING_MS = 2200

const isUnlocked = ref(props.unlocked)
const showLoading = ref(!props.unlocked) // lewati loading kalau kembali dari Fase 4
let loadingTimer = null

if (showLoading.value) {
  loadingTimer = setTimeout(() => (showLoading.value = false), LOADING_MS)
}

function onUnlocked() {
  isUnlocked.value = true
  emit('update:unlocked', true)
}

// Tombol Exit: reset semua state ke layar sandi dan matikan musik
function onExit() {
  emit('stop-music')
  isUnlocked.value = false
  emit('update:unlocked', false)
}

onBeforeUnmount(() => clearTimeout(loadingTimer))
</script>

<style scoped>
/* ================= TRANSISI ANTAR LANGKAH ================= */
.landing-enter-active,
.landing-leave-active {
  transition: opacity 0.6s ease;
}

.landing-enter-from,
.landing-leave-to {
  opacity: 0;
}

/* ================= LOADING ================= */
.loading {
  position: fixed;
  inset: 0;
  z-index: 10;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 22px;
  padding: 24px;
  background: transparent;
  font-family: 'Cormorant Garamond', 'Playfair Display', Georgia, serif;
}

.loading-flower {
  width: 64px;
  height: 64px;
  filter: drop-shadow(0 0 14px rgba(255, 105, 170, 0.75));
  animation: bloom 2.2s ease-in-out infinite;
}

.loading-text {
  margin: 0;
  font-size: 0.95rem;
  letter-spacing: 1.4px;
  color: rgba(248, 220, 232, 0.6);
  animation: fadePulse 2.2s ease-in-out infinite;
}

/* ================= KONTEN (SCROLL VERTIKAL) ================= */
.landing-scroll {
  position: fixed;
  inset: 0;
  z-index: 10;
  overflow-x: hidden;
  overflow-y: auto;
  overscroll-behavior: contain;
  -webkit-overflow-scrolling: touch;
  scroll-behavior: smooth;
  scrollbar-width: thin;
  scrollbar-color: rgba(255, 166, 201, 0.35) transparent;
  /* tint wine tipis supaya bintang dari App.vue tetap samar terlihat */
  background: linear-gradient(
    180deg,
    rgba(52, 12, 34, 0.5) 0%,
    rgba(70, 16, 46, 0.58) 55%,
    rgba(40, 8, 28, 0.62) 100%
  );
}

/* Konten selalu di depan bunga melayang (z-index 0) */
.landing-content {
  position: relative;
  z-index: 1;
}

@keyframes bloom {
  0%,
  100% {
    transform: scale(0.92) rotate(-6deg);
  }
  50% {
    transform: scale(1.1) rotate(8deg);
  }
}

@keyframes fadePulse {
  0%,
  100% {
    opacity: 0.4;
  }
  50% {
    opacity: 1;
  }
}

@media (prefers-reduced-motion: reduce) {
  .loading-flower,
  .loading-text {
    animation: none;
  }

  .landing-scroll {
    scroll-behavior: auto;
  }
}
</style>