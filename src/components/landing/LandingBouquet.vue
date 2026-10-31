<!-- LandingBouquet.vue : Hero + Buket Digital + Surat (scroll) -->
<template>
  <div ref="rootRef" class="bouquet-page">
    <!-- ================= HERO ================= -->
    <section class="hero">
      <p class="eyebrow">✿ Your Special Day ✿</p>
      <h1 class="hero-title">
        <span>Happy</span>
        <em>Birthday</em>
        <span>{{ recipientName }}</span>
      </h1>
      <div class="hero-line"></div>
      <p class="hero-sub">The most special day</p>
      <div class="scroll-hint" aria-hidden="true">
        <span>scroll</span>
        <svg viewBox="0 0 24 24"><path d="M6 9l6 6 6-6" /></svg>
      </div>
    </section>

    <!-- ================= BUKET DIGITAL ================= -->
    <section class="bouquet-section">
      <p class="eyebrow" data-reveal>— My first gift —</p>
      <h2 class="section-title" data-reveal>A Digital Bouquet</h2>
      <p class="section-sub" data-reveal>Each flower holds a little message just for you</p>

      <div class="stage" data-reveal>
        <!-- Batang + pot -->
        <svg class="plant" viewBox="0 0 200 150" aria-hidden="true">
          <g fill="none" stroke="#4f9a5b" stroke-width="3.2" stroke-linecap="round">
            <path d="M100 100 C100 70 100 40 100 10" />
            <path d="M100 100 C92 78 74 44 62 22" />
            <path d="M100 100 C108 78 126 44 138 22" />
            <path d="M100 100 C86 92 70 80 52 74" stroke-width="2.6" />
            <path d="M100 100 C114 92 130 80 148 74" stroke-width="2.6" />
          </g>
          <g fill="#5fb46d">
            <ellipse cx="82" cy="70" rx="13" ry="6" transform="rotate(-38 82 70)" />
            <ellipse cx="118" cy="70" rx="13" ry="6" transform="rotate(38 118 70)" />
            <ellipse cx="66" cy="84" rx="12" ry="5.5" transform="rotate(-18 66 84)" />
            <ellipse cx="134" cy="84" rx="12" ry="5.5" transform="rotate(18 134 84)" />
          </g>
          <path d="M62 100 H138 L128 144 Q127 148 123 148 H77 Q73 148 72 144 Z" fill="#e8628f" />
          <rect x="58" y="94" width="84" height="14" rx="7" fill="#f58db1" />
          <path d="M70 118 Q100 126 130 118" stroke="#ffffff55" stroke-width="2" fill="none" />
        </svg>

        <!-- Bunga yang bisa diklik -->
        <button
          v-for="(f, i) in flowers"
          :key="i"
          class="flower"
          :class="{ 'is-active': activeIndex === i, 'is-read': read.includes(i) }"
          type="button"
          :aria-label="`Bunga ${i + 1}, ketuk untuk membaca pesan`"
          :style="{
            left: f.x + '%',
            top: f.y + '%',
            fontSize: f.size + 'rem',
            '--delay': f.delay + 's'
          }"
          @click="pickFlower(i)"
        >
          <span class="flower-inner">
            <!-- Edelweis: digambar dengan SVG karena tidak ada emojinya -->
            <svg
              v-if="f.type === 'edelweiss'"
              class="edelweiss"
              viewBox="0 0 64 64"
              aria-hidden="true"
            >
              <!-- kelopak bawah (lebih panjang) -->
              <path
                v-for="n in 8"
                :key="'b' + n"
                d="M32 32 C26.5 22 27 10 32 2 C37 10 37.5 22 32 32 Z"
                fill="#f4f1e6"
                stroke="#d8d2bd"
                stroke-width="0.7"
                :transform="`rotate(${(n - 1) * 45} 32 32)`"
              />
              <!-- kelopak atas (lebih pendek, putih) -->
              <path
                v-for="n in 8"
                :key="'t' + n"
                d="M32 32 C28.5 25 28.8 17 32 11 C35.2 17 35.5 25 32 32 Z"
                fill="#ffffff"
                :transform="`rotate(${(n - 1) * 45 + 22.5} 32 32)`"
              />
              <!-- bunga kecil kuning di tengah -->
              <circle cx="32" cy="32" r="5.2" fill="#f2cf5b" />
              <circle cx="29.6" cy="30.4" r="1.7" fill="#fbe596" />
              <circle cx="34.2" cy="30.8" r="1.5" fill="#fbe596" />
              <circle cx="32.2" cy="34.6" r="1.6" fill="#e3b93f" />
            </svg>
            <template v-else>{{ f.emoji }}</template>
          </span>
        </button>

        <!-- Percikan saat bunga diklik -->
        <span
          v-for="p in sparks"
          :key="p.id"
          class="spark"
          :style="{ left: p.x + '%', top: p.y + '%', '--dx': p.dx + 'px', '--dy': p.dy + 'px' }"
          @animationend="removeSpark(p.id)"
        >
          {{ p.char }}
        </span>
      </div>

      <!-- Pesan kecil (fade-in) -->
      <div class="message-card" aria-live="polite">
        <Transition name="msg" mode="out-in">
          <p v-if="activeIndex === -1" key="idle" class="message idle">
            Ketuk salah satu bunga ✿
          </p>
          <p v-else :key="activeIndex" class="message">{{ flowers[activeIndex].message }}</p>
        </Transition>
      </div>
      <p class="progress">{{ read.length }} / {{ flowers.length }} pesan terbaca</p>
    </section>

    <!-- ================= SURAT ================= -->
    <section class="letter-section">
      <p class="eyebrow" data-reveal>— From my heart —</p>
      <h2 class="section-title" data-reveal>A Letter For You</h2>

      <article class="letter" data-reveal>
        <div class="letter-ornament" aria-hidden="true">✿ ✦ ✿</div>

        <p class="letter-to" data-reveal>To my beautiful {{ recipientName }},</p>

        <p
          v-for="(para, i) in letter"
          :key="i"
          class="letter-p"
          :class="{ italic: para.italic, ps: para.ps }"
          data-reveal
        >
          {{ para.text }}
        </p>

        <div class="letter-sign" data-reveal>
          <p>With all my love,</p>
          <strong>{{ senderName }}</strong>
        </div>
      </article>
    </section>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'

/* ================= EDIT DI SINI ================= */
const recipientName = 'Adisti Naura' // nama penerima
const senderName = 'Aqeel Gantneng' // nama pengirim (tanda tangan surat)

// 9 bunga. x / y dalam persen dari area buket. Ganti message sesuka hati.
// Bunga dengan type: 'edelweiss' digambar pakai SVG (bukan emoji).
const flowers = [
  {
    emoji: '🌻',
    x: 50,
    y: 63,
    size: 3.1,
    delay: 0,
    message: 'You are my sunshine. Even the cloudiest days feel warm when you are near.'
  },
  {
    emoji: '🏵️',
    x: 37,
    y: 70,
    size: 2.3,
    delay: 0.6,
    message: 'Your laughter is the most beautiful sound I have ever heard.'
  },
  {
    emoji: '🌹',
    x: 63,
    y: 70,
    size: 2.4,
    delay: 1.1,
    message:
      'You are the most beautiful rose that ever bloomed, full of love and a beauty no one can match.'
  },
  {
    emoji: '🌷',
    x: 24,
    y: 27,
    size: 2.4,
    delay: 0.3,
    message: 'The gentleness of your heart makes the world around you a better place.'
  },
  {
    emoji: '🌸',
    x: 77,
    y: 22,
    size: 2.4,
    delay: 0.9,
    message: 'You are as lovely as a cherry blossom, sweet, and you bring joy wherever you go.'
  },
  {
    emoji: '🌼',
    x: 14,
    y: 48,
    size: 2.1,
    delay: 1.4,
    message: 'Your cheerfulness is like a daisy, simple, yet it always brightens my day.'
  },
  {
    emoji: '🌺',
    x: 87,
    y: 46,
    size: 2.2,
    delay: 0.2,
    message: 'Thank you for being you. I am so grateful to have you in my life.'
  },
  // ----- BUNGA BARU 1: Edelweis (SVG) -----
  {
    type: 'edelweiss',
    x: 50,
    y: 37,
    size: 2.6,
    delay: 0.45,
    message:
      'Like an edelweiss on a mountain peak, you are rare, precious, and worth every step of the journey. My love for you stays true in every season.'
  },
  // ----- BUNGA BARU 2: Teratai (kalau tampil kotak di HP lama, ganti emojinya ke 💮) -----
  {
    emoji: '🪷',
    x: 70,
    y: 49,
    size: 2.3,
    delay: 0.7,
    message:
      'Like a lotus rising beautifully from the water, you grow stronger and more graceful every day. I am so proud of you.'
  }
]

// Isi surat (boleh ditambah / dikurangi / diganti sesukamu)
const letter = [
  {
    text: 'On this special day, I want you to know that every day with you is a gift beyond measure. You bring light into every corner of my life, even on the darkest days.'
  },
  {
    text: 'Your laughter is the most beautiful music I have ever heard. The way you see the world, with sincerity and kindness, makes me want to be a better person every single day.'
  },
  {
    text: 'Thank you for being yourself, with all your uniqueness, your gentle heart, and your spirit that never fades. You are the most beautiful gift life has ever given me.',
    italic: true
  },
  {
    text: 'On your birthday, I hope every dream of yours comes true, every prayer is answered, and your happiness grows beyond measure. You deserve all the beautiful things in this world.'
  },
  {
    text: 'P.S. I will always be here for you, today, tomorrow, and forever. 💕',
    ps: true
  }
]
/* ================================================= */

const rootRef = ref(null)

// ---------- Bunga & pesan ----------
const activeIndex = ref(-1)
const read = ref([])
const sparks = ref([])
let sparkId = 0
const SPARK_CHARS = ['🌸', '✨', '💖', '🌷', '✨']

function pickFlower(i) {
  activeIndex.value = i
  if (!read.value.includes(i)) read.value.push(i)

  const f = flowers[i]
  for (let k = 0; k < 7; k++) {
    const a = (Math.PI * 2 * k) / 7 + Math.random() * 0.6
    const d = 34 + Math.random() * 34
    sparks.value.push({
      id: sparkId++,
      x: f.x,
      y: f.y,
      dx: Math.round(Math.cos(a) * d),
      dy: Math.round(Math.sin(a) * d - 14),
      char: SPARK_CHARS[k % SPARK_CHARS.length]
    })
  }
}

function removeSpark(id) {
  sparks.value = sparks.value.filter((p) => p.id !== id)
}

// ---------- Intersection Observer: muncul (fade-in up) saat di-scroll ----------
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
          io.unobserve(entry.target) // cukup sekali
        }
      })
    },
    { threshold: 0.15, rootMargin: '0px 0px -6% 0px' }
  )
  els.forEach((el) => io.observe(el))
})

onBeforeUnmount(() => io?.disconnect())
</script>

<style scoped>
.bouquet-page {
  --ff-serif: 'Cormorant Garamond', 'Playfair Display', Georgia, serif;
  --pink: var(--neon-pink, #ff1493);
  --blush: #f8dce8;
  --rose: #ffa6c9;

  width: 100%;
  max-width: 620px;
  margin: 0 auto;
  padding: 0 20px;
  box-sizing: border-box;
  font-family: var(--ff-serif);
  color: var(--blush);
  text-align: center;
}

/* ================= ANIMASI MUNCUL (IntersectionObserver) ================= */
[data-reveal] {
  opacity: 0;
  transform: translateY(26px);
  transition: opacity 0.9s ease, transform 0.9s cubic-bezier(0.2, 0.8, 0.25, 1);
}

[data-reveal].is-visible {
  opacity: 1;
  transform: translateY(0);
}

/* "Your Special Day", "My first gift", "From my heart" */
.eyebrow {
  margin: 0 0 14px;
  font-size: 0.95rem;
  font-weight: 500;
  letter-spacing: 3px;
  text-transform: uppercase;
  color: rgba(255, 194, 220, 0.95);
  text-shadow: 0 0 12px rgba(255, 105, 170, 0.35);
}

.section-title {
  margin: 0 0 10px;
  font-size: clamp(2rem, 8.5vw, 2.8rem);
  font-weight: 400;
  color: #fbe7ef;
  text-shadow: 0 0 20px rgba(255, 105, 170, 0.3);
}

/* "Each flower holds a little message just for you" */
.section-sub {
  margin: 0 0 8px;
  font-size: 1.15rem;
  font-weight: 500;
  color: rgba(248, 220, 232, 0.9);
}

/* ================= HERO ================= */
.hero {
  min-height: 100svh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 40px 0 70px;
  animation: heroIn 1.4s ease both;
}

.hero-title {
  margin: 8px 0 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  font-weight: 400;
  line-height: 1.08;
  font-size: clamp(3rem, 15vw, 4.6rem);
  color: #fff3f7;
  text-shadow: 0 0 26px rgba(255, 105, 170, 0.35);
}

.hero-title em {
  font-style: italic;
  color: var(--rose);
}

.hero-line {
  width: 90px;
  height: 1px;
  margin: 26px 0 14px;
  background: linear-gradient(90deg, transparent, var(--rose), transparent);
}

/* "The most special day" */
.hero-sub {
  margin: 0;
  font-size: 1rem;
  font-weight: 500;
  letter-spacing: 3px;
  text-transform: uppercase;
  color: rgba(248, 220, 232, 0.9);
  text-shadow: 0 0 10px rgba(255, 105, 170, 0.25);
}

.scroll-hint {
  margin-top: 54px;
  display: flex;
  flex-direction: column;
  align-items: center;
  font-size: 0.7rem;
  letter-spacing: 3px;
  text-transform: uppercase;
  color: rgba(248, 220, 232, 0.5);
  animation: bob 1.8s ease-in-out infinite;
}

.scroll-hint svg {
  width: 22px;
  height: 22px;
  fill: none;
  stroke: currentColor;
  stroke-width: 1.8;
  stroke-linecap: round;
  stroke-linejoin: round;
}

/* ================= BUKET ================= */
.bouquet-section {
  padding: 50px 0 40px;
}

.stage {
  position: relative;
  width: min(92%, 380px);
  aspect-ratio: 5 / 6;
  margin: 12px auto 0;
}

.plant {
  position: absolute;
  left: 50%;
  bottom: 0;
  width: 58%;
  transform: translateX(-50%);
  filter: drop-shadow(0 8px 18px rgba(0, 0, 0, 0.35));
  pointer-events: none;
}

.flower {
  position: absolute;
  z-index: 2;
  margin: 0;
  padding: 10px;
  border: none;
  background: transparent;
  line-height: 1;
  cursor: pointer;
  transform: translate(-50%, -50%);
  -webkit-tap-highlight-color: transparent;
}

.flower-inner {
  display: block;
  animation: floaty 3.4s ease-in-out infinite;
  animation-delay: var(--delay, 0s);
  filter: drop-shadow(0 0 8px rgba(255, 105, 170, 0.35));
  transition: transform 0.25s ease, filter 0.25s ease;
}

/* Edelweis (SVG) mengikuti ukuran font tombolnya (1em) */
.edelweiss {
  display: block;
  width: 1em;
  height: 1em;
  overflow: visible;
}

.flower:hover .flower-inner {
  transform: scale(1.18);
}

.flower:active .flower-inner {
  transform: scale(0.92);
}

.flower.is-read .flower-inner {
  filter: drop-shadow(0 0 10px rgba(255, 190, 215, 0.8));
}

.flower.is-active .flower-inner {
  transform: scale(1.3);
  filter: drop-shadow(0 0 16px var(--pink));
}

.flower:focus-visible {
  outline: 2px solid var(--rose);
  border-radius: 50%;
}

.spark {
  position: absolute;
  z-index: 3;
  font-size: 1rem;
  pointer-events: none;
  transform: translate(-50%, -50%);
  animation: spark 0.95s ease-out forwards;
}

.message-card {
  min-height: 110px;
  width: min(100%, 400px);
  margin: 18px auto 0;
  padding: 18px 22px;
  box-sizing: border-box;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 20px;
  border: 1px solid rgba(255, 166, 201, 0.18);
  background: rgba(255, 255, 255, 0.06);
  backdrop-filter: blur(4px);
  -webkit-backdrop-filter: blur(4px);
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.25);
}

.message {
  margin: 0;
  font-size: 1.12rem;
  font-style: italic;
  line-height: 1.6;
  color: #f6d5e1;
}

.message.idle {
  color: rgba(248, 220, 232, 0.5);
}

/* "0 / 9 pesan terbaca" */
.progress {
  margin: 12px 0 0;
  font-size: 1rem;
  font-weight: 500;
  letter-spacing: 1.5px;
  color: rgba(248, 220, 232, 0.85);
}

.msg-enter-active,
.msg-leave-active {
  transition: opacity 0.45s ease, transform 0.45s ease;
}

.msg-enter-from {
  opacity: 0;
  transform: translateY(8px);
}

.msg-leave-to {
  opacity: 0;
  transform: translateY(-6px);
}

/* ================= SURAT ================= */
.letter-section {
  padding: 70px 0 60px;
}

.letter {
  margin-top: 24px;
  padding: 30px 24px 28px;
  border-radius: 20px;
  text-align: left;
  color: #5e2c42;
  background: linear-gradient(180deg, #fff3f7 0%, #fde3ec 100%);
  box-shadow: 0 0 40px rgba(255, 105, 170, 0.22), 0 24px 50px rgba(0, 0, 0, 0.4);
}

.letter-ornament {
  margin-bottom: 22px;
  text-align: center;
  font-size: 0.85rem;
  letter-spacing: 6px;
  color: #e58db0;
}

.letter-to {
  margin: 0 0 18px;
  font-size: 1.55rem;
  font-weight: 500;
  color: #4a1f33;
}

.letter-p {
  margin: 0 0 18px;
  font-size: 1.1rem;
  line-height: 1.85;
}

.letter-p.italic {
  font-style: italic;
  color: #8a4a66;
}

.letter-p.ps {
  padding: 14px 16px;
  border-radius: 12px;
  font-style: italic;
  color: #8a4a66;
  background: rgba(240, 111, 164, 0.12);
}

.letter-sign {
  margin-top: 26px;
  text-align: right;
}

.letter-sign p {
  margin: 0 0 2px;
  font-style: italic;
  color: #9a5a76;
}

.letter-sign strong {
  font-size: 1.7rem;
  font-weight: 500;
  color: #4a1f33;
}

/* ================= KEYFRAMES ================= */
@keyframes heroIn {
  from {
    opacity: 0;
    transform: translateY(18px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes bob {
  0%,
  100% {
    transform: translateY(0);
    opacity: 0.45;
  }
  50% {
    transform: translateY(7px);
    opacity: 1;
  }
}

@keyframes floaty {
  0%,
  100% {
    transform: translateY(0) rotate(-3deg);
  }
  50% {
    transform: translateY(-7px) rotate(3deg);
  }
}

@keyframes spark {
  0% {
    opacity: 1;
    transform: translate(-50%, -50%) scale(0.5);
  }
  100% {
    opacity: 0;
    transform: translate(calc(-50% + var(--dx)), calc(-50% + var(--dy))) scale(1.25);
  }
}

@media (min-width: 700px) {
  .letter {
    padding: 40px 44px 36px;
  }

  .letter-p {
    font-size: 1.18rem;
  }
}

@media (prefers-reduced-motion: reduce) {
  .hero,
  .scroll-hint,
  .flower-inner {
    animation: none;
  }

  [data-reveal] {
    transition: opacity 0.3s ease;
    transform: none;
  }
}
</style>