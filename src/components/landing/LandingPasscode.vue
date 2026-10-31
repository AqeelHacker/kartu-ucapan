<!-- LandingPasscode.vue : Layar kunci (Fase 0, langkah 1) -->
<template>
  <section class="passcode" aria-label="Layar kunci">
    <div
      class="card"
      :class="{
        'is-error': status === 'error',
        'is-success': status === 'success',
        'is-angry': status === 'angry'
      }"
    >
      <!-- Ikon bunga di tepi atas kartu -->
      <svg class="flower" viewBox="0 0 64 64" aria-hidden="true">
        <defs>
          <radialGradient id="pc-petal" cx="50%" cy="30%" r="80%">
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
          fill="url(#pc-petal)"
          :transform="`rotate(${(i - 1) * 72} 32 32)`"
        />
        <circle cx="32" cy="32" r="6.5" fill="#ffe3ef" />
      </svg>

      <h1 class="title">For You, My Love</h1>

      <!-- Subjudul berubah jadi omelan saat tombol ✕ ditekan -->
      <p class="subtitle" :class="{ angry: status === 'angry' }" aria-live="polite">
        <Transition name="sub" mode="out-in">
          <span :key="status === 'angry' ? 'angry' : 'normal'">
            {{ status === 'angry' ? ANGRY_TEXT : 'Enter our secret code' }}
          </span>
        </Transition>
      </p>

      <!-- Indikator 4 titik -->
      <div
        class="dots"
        :class="{ shake: status === 'error' }"
        role="status"
        :aria-label="`${code.length} of ${CODE_LENGTH} digits entered`"
      >
        <span
          v-for="n in CODE_LENGTH"
          :key="n"
          class="dot"
          :class="{ filled: n <= code.length }"
        ></span>
      </div>

      <!-- Keypad -->
      <div class="pad" :class="{ jolt }">
        <button
          v-for="d in ['1', '2', '3', '4', '5', '6', '7', '8', '9']"
          :key="d"
          class="key"
          :class="{ 'is-pressed': pressed === d }"
          type="button"
          @click="press(d)"
        >
          {{ d }}
        </button>

        <!-- ✕ : bikin keypad "marah" -->
        <button
          class="key key-icon"
          :class="{ 'is-pressed': pressed === 'clear' }"
          type="button"
          aria-label="Clear"
          @click="angryClear"
        >
          ✕
        </button>
        <button
          class="key"
          :class="{ 'is-pressed': pressed === '0' }"
          type="button"
          @click="press('0')"
        >
          0
        </button>
        <button
          class="key key-icon"
          :class="{ 'is-pressed': pressed === 'back' }"
          type="button"
          aria-label="Delete last digit"
          @click="backspace"
        >
          <svg viewBox="0 0 24 24" aria-hidden="true">
            <path d="M21 5H9l-6 7 6 7h12a1 1 0 0 0 1-1V6a1 1 0 0 0-1-1z" />
            <path d="M14 9.5l5 5M19 9.5l-5 5" />
          </svg>
        </button>
      </div>

      <p class="hint">Hint: our next aniv 💕</p>
    </div>
  </section>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'

/*
  Emit:
  - 'play-music' : dikirim SEKALIGUS saat digit ke-4 benar (masih di dalam event klik,
                   supaya browser mengizinkan audio autoplay, termasuk iOS Safari)
  - 'unlocked'   : dikirim sedikit kemudian, setelah animasi sukses selesai
*/
const emit = defineEmits(['play-music', 'unlocked'])

/* ================= EDIT DI SINI ================= */
const SECRET_CODE = '310826' // sandi
const KEY_SFX_SRC = '/audio/beep.mp3' // sound effect saat tombol ditekan (ganti nama file sesuai punyamu)
const KEY_SFX_VOLUME = 0.6 // 0 - 1
const ANGRY_TEXT = "Why did you press that? Don't you want to see the surprise? 😡"
/* ================================================= */

const CODE_LENGTH = SECRET_CODE.length
const SUCCESS_DELAY_MS = 750
const ERROR_RESET_MS = 700

const code = ref('')
const status = ref('idle') // 'idle' | 'error' | 'success' | 'angry'
const pressed = ref('')
const jolt = ref(false)
const timers = []

function flash(key) {
  pressed.value = key
  timers.push(setTimeout(() => (pressed.value = ''), 220))
}

// Sound effect: objek Audio baru tiap tekan supaya ketikan cepat tidak saling memotong
function playSfx() {
  if (!KEY_SFX_SRC) return
  try {
    const s = new Audio(KEY_SFX_SRC)
    s.volume = KEY_SFX_VOLUME
    s.play().catch(() => {})
  } catch (e) {
    /* abaikan kalau audio tidak tersedia */
  }
}

function press(d) {
  if (status.value === 'success' || status.value === 'error') return
  if (code.value.length >= CODE_LENGTH) return

  // Klik angka setelah keypad marah -> langsung normal lagi
  if (status.value === 'angry') status.value = 'idle'

  flash(d)
  playSfx()
  code.value += d
  if (code.value.length < CODE_LENGTH) return

  if (code.value === SECRET_CODE) {
    status.value = 'success'
    emit('play-music') // sinkron dengan klik pengguna
    timers.push(setTimeout(() => emit('unlocked'), SUCCESS_DELAY_MS))
  } else {
    status.value = 'error'
    if (navigator.vibrate) navigator.vibrate(180)
    timers.push(
      setTimeout(() => {
        code.value = ''
        status.value = 'idle'
      }, ERROR_RESET_MS)
    )
  }
}

function backspace() {
  if (status.value === 'success' || status.value === 'error') return
  flash('back')
  playSfx()
  code.value = code.value.slice(0, -1)
}

// Tombol ✕: hapus isian + keypad marah (aura merah + omelan) sampai user menekan angka lagi
function angryClear() {
  if (status.value === 'success' || status.value === 'error') return
  flash('clear')
  playSfx()
  code.value = ''
  status.value = 'angry'
  if (navigator.vibrate) navigator.vibrate([60, 40, 60])

  jolt.value = true
  timers.push(setTimeout(() => (jolt.value = false), 500))
}

function onKeydown(e) {
  if (/^[0-9]$/.test(e.key)) press(e.key)
  else if (e.key === 'Backspace') backspace()
  else if (e.key === 'Escape') code.value = ''
}

onMounted(() => window.addEventListener('keydown', onKeydown))
onBeforeUnmount(() => {
  window.removeEventListener('keydown', onKeydown)
  timers.forEach(clearTimeout)
})
</script>

<style scoped>
.passcode {
  --ff-serif: 'Cormorant Garamond', 'Playfair Display', Georgia, serif;
  --pink: var(--neon-pink, #ff1493);
  --rose: #ffa6c9;
  --blush: #f8dce8;
  --angry: #ff3b3b;

  position: fixed;
  inset: 0;
  z-index: 10;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 24px 16px;
  box-sizing: border-box;
  background: transparent; /* bintang dari App.vue tetap terlihat */
  font-family: var(--ff-serif);
  color: var(--blush);
}

/* ================= KARTU KACA ================= */
.card {
  position: relative;
  width: min(88vw, 340px);
  box-sizing: border-box;
  padding: 58px 22px 20px;
  text-align: center;
  border-radius: 28px;
  border: 1px solid rgba(255, 166, 201, 0.2);
  background: linear-gradient(160deg, rgba(130, 40, 85, 0.3), rgba(60, 14, 40, 0.42));
  backdrop-filter: blur(5px);
  -webkit-backdrop-filter: blur(5px);
  box-shadow: 0 24px 60px rgba(0, 0, 0, 0.5), inset 0 0 44px rgba(255, 20, 147, 0.07);
  animation: cardIn 0.9s cubic-bezier(0.2, 0.9, 0.25, 1) both;
  transition: box-shadow 0.5s ease, transform 0.5s ease, border-color 0.35s ease,
    background 0.35s ease;
}

.card.is-success {
  transform: scale(0.98);
  box-shadow: 0 0 46px rgba(255, 20, 147, 0.55), 0 24px 60px rgba(0, 0, 0, 0.5),
    inset 0 0 60px rgba(255, 20, 147, 0.18);
}

/* ================= MODE MARAH ================= */
.card.is-angry {
  border-color: rgba(255, 70, 70, 0.55);
  background: linear-gradient(160deg, rgba(150, 25, 40, 0.38), rgba(70, 8, 18, 0.5));
  animation: cardIn 0.9s cubic-bezier(0.2, 0.9, 0.25, 1) both, angryGlow 1.1s ease-in-out infinite alternate;
}

.is-angry .title {
  text-shadow: 0 0 18px rgba(255, 60, 60, 0.55);
}

.is-angry .flower {
  filter: drop-shadow(0 0 12px rgba(255, 60, 60, 0.85)) hue-rotate(-30deg) saturate(1.4);
}

.flower {
  position: absolute;
  top: -34px;
  left: 50%;
  width: 68px;
  height: 68px;
  margin-left: -34px;
  filter: drop-shadow(0 0 12px rgba(255, 105, 170, 0.7));
  animation: flowerFloat 3.6s ease-in-out infinite;
}

.title {
  margin: 0;
  font-size: clamp(1.9rem, 8vw, 2.3rem);
  font-weight: 500;
  letter-spacing: 0.4px;
  color: #fbe7ef;
  text-shadow: 0 0 18px rgba(255, 105, 170, 0.35);
  transition: text-shadow 0.35s ease;
}

.subtitle {
  /* tinggi tetap supaya tata letak tidak bergeser saat teks omelan muncul */
  min-height: 2.8em;
  margin: 6px 0 14px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1rem;
  line-height: 1.35;
  letter-spacing: 0.8px;
  color: rgba(248, 220, 232, 0.75);
}

.subtitle.angry {
  font-size: 0.98rem;
  font-style: italic;
  letter-spacing: 0.2px;
  color: #ff9a9a;
  text-shadow: 0 0 12px rgba(255, 50, 50, 0.7);
}

.sub-enter-active,
.sub-leave-active {
  transition: opacity 0.22s ease, transform 0.22s ease;
}

.sub-enter-from,
.sub-leave-to {
  opacity: 0;
  transform: translateY(5px);
}

/* ================= INDIKATOR ================= */
.dots {
  display: flex;
  justify-content: center;
  gap: 16px;
  margin-bottom: 24px;
}

.dot {
  width: 12px;
  height: 12px;
  box-sizing: border-box;
  border-radius: 50%;
  border: 1.5px solid rgba(255, 166, 201, 0.5);
  background: transparent;
  transition: background 0.2s ease, box-shadow 0.2s ease, transform 0.2s ease, border-color 0.2s ease;
}

.dot.filled {
  background: var(--rose);
  border-color: var(--rose);
  box-shadow: 0 0 10px rgba(255, 150, 195, 0.9);
  transform: scale(1.12);
}

.is-error .dot.filled {
  background: #ff4d6d;
  border-color: #ff4d6d;
  box-shadow: 0 0 12px rgba(255, 77, 109, 0.95);
}

.is-success .dot.filled {
  background: var(--pink);
  border-color: var(--pink);
  box-shadow: 0 0 16px var(--pink);
}

.is-angry .dot {
  border-color: rgba(255, 90, 90, 0.6);
}

.shake {
  animation: shake 0.45s ease;
}

/* ================= KEYPAD ================= */
.pad {
  position: relative;
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  justify-items: center;
  gap: 14px 12px;
}

/* Aura merah di belakang keypad */
.pad::before {
  content: '';
  position: absolute;
  inset: -20px -16px;
  border-radius: 36px;
  background: radial-gradient(
    ellipse at center,
    rgba(255, 30, 30, 0.42) 0%,
    rgba(255, 0, 0, 0.16) 55%,
    transparent 78%
  );
  filter: blur(8px);
  opacity: 0;
  transition: opacity 0.35s ease;
  pointer-events: none;
}

.is-angry .pad::before {
  opacity: 1;
  animation: auraPulse 1s ease-in-out infinite alternate;
}

.pad.jolt {
  animation: jolt 0.5s ease;
}

.key {
  position: relative;
  z-index: 1;
  width: clamp(54px, 17vw, 64px);
  height: clamp(54px, 17vw, 64px);
  padding: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  border: 1px solid rgba(255, 166, 201, 0.16);
  background: rgba(255, 255, 255, 0.055);
  color: #f6d5e1;
  font-family: var(--ff-serif);
  font-size: 1.55rem;
  cursor: pointer;
  user-select: none;
  touch-action: manipulation;
  -webkit-tap-highlight-color: transparent;
  transition: transform 0.15s ease, box-shadow 0.2s ease, border-color 0.2s ease,
    background 0.2s ease, color 0.2s ease;
}

.key:hover {
  border-color: rgba(255, 166, 201, 0.4);
}

.is-angry .key {
  border-color: rgba(255, 80, 80, 0.55);
  background: rgba(255, 40, 40, 0.1);
  color: #ffc9c9;
  box-shadow: 0 0 14px rgba(255, 40, 40, 0.45), inset 0 0 12px rgba(255, 40, 40, 0.18);
}

/* Efek glow pink saat ditekan */
.key:active,
.key.is-pressed {
  transform: scale(0.93);
  border-color: var(--pink);
  background: rgba(255, 20, 147, 0.2);
  box-shadow: 0 0 20px var(--pink), inset 0 0 14px rgba(255, 20, 147, 0.45);
  color: #fff;
}

.is-angry .key:active,
.is-angry .key.is-pressed {
  border-color: var(--angry);
  background: rgba(255, 30, 30, 0.3);
  box-shadow: 0 0 22px var(--angry), inset 0 0 14px rgba(255, 30, 30, 0.5);
}

.key-icon {
  font-size: 1.05rem;
  color: rgba(246, 213, 225, 0.75);
}

.key-icon svg {
  width: 24px;
  height: 24px;
  fill: none;
  stroke: currentColor;
  stroke-width: 1.7;
  stroke-linecap: round;
  stroke-linejoin: round;
}

.hint {
  margin: 22px 0 2px;
  font-size: 0.88rem;
  letter-spacing: 0.8px;
  color: rgba(248, 220, 232, 0.6);
}

/* ================= KEYFRAMES ================= */
@keyframes cardIn {
  from {
    opacity: 0;
    transform: translateY(26px) scale(0.97);
  }
  to {
    opacity: 1;
    transform: translateY(0) scale(1);
  }
}

@keyframes flowerFloat {
  0%,
  100% {
    transform: translateY(0) rotate(-4deg);
  }
  50% {
    transform: translateY(-5px) rotate(5deg);
  }
}

@keyframes shake {
  0%,
  100% {
    transform: translateX(0);
  }
  18% {
    transform: translateX(-9px);
  }
  36% {
    transform: translateX(8px);
  }
  54% {
    transform: translateX(-6px);
  }
  72% {
    transform: translateX(4px);
  }
}

@keyframes jolt {
  0%,
  100% {
    transform: translate(0, 0) rotate(0);
  }
  15% {
    transform: translate(-6px, 1px) rotate(-1.5deg);
  }
  35% {
    transform: translate(6px, -1px) rotate(1.5deg);
  }
  55% {
    transform: translate(-4px, 1px) rotate(-1deg);
  }
  75% {
    transform: translate(4px, 0) rotate(1deg);
  }
}

@keyframes angryGlow {
  from {
    box-shadow: 0 0 26px rgba(255, 40, 40, 0.4), 0 24px 60px rgba(0, 0, 0, 0.5),
      inset 0 0 40px rgba(255, 0, 0, 0.14);
  }
  to {
    box-shadow: 0 0 60px rgba(255, 40, 40, 0.8), 0 24px 60px rgba(0, 0, 0, 0.5),
      inset 0 0 70px rgba(255, 0, 0, 0.28);
  }
}

@keyframes auraPulse {
  from {
    opacity: 0.65;
    transform: scale(0.98);
  }
  to {
    opacity: 1;
    transform: scale(1.04);
  }
}

@media (max-height: 640px) {
  .card {
    padding-top: 50px;
  }

  .pad {
    gap: 10px 12px;
  }

  .key {
    width: 50px;
    height: 50px;
    font-size: 1.35rem;
  }

  .dots {
    margin-bottom: 16px;
  }
}

@media (prefers-reduced-motion: reduce) {
  .card,
  .flower,
  .shake,
  .pad.jolt,
  .is-angry .pad::before {
    animation: none;
  }
}
</style>