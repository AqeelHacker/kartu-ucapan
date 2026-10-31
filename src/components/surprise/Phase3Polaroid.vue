<!-- Phase3Polaroid.vue -->
<template>
  <section class="phase3" :class="{ 'is-leaving': isLeaving }">
    <!-- ================= TEXTBOX LEBAR ================= -->
    <div class="message-box">
      <p class="message-text" :key="displayMessage">{{ displayMessage }}</p>
    </div>

    <!-- ================= BUKU FOTO 3D ================= -->
    <div class="book-scene" role="group" aria-label="Buku foto">
      <div class="book-shadow" :class="{ 'is-open': isOpen }"></div>

      <div class="book" :class="{ 'is-open': isOpen }">
        <!-- Papan alas buku (terlihat saat terbuka) -->
        <div class="book-board"></div>

        <!-- Halaman dasar: sisi kanan spread terakhir -->
        <div class="face base-page">
          <figure class="polaroid">
            <img
              :src="lastRightImage"
              alt="Foto halaman terakhir"
              draggable="false"
              @error="onImgError"
            />
          </figure>
        </div>

        <!-- Lembaran (leaf): depan = sisi kanan, belakang = sisi kiri -->
        <div
          v-for="(leaf, k) in leaves"
          :key="k"
          class="leaf"
          :style="leafStyle(k)"
        >
          <!-- Sisi depan (tampil di kanan sebelum dibalik) -->
          <div
            class="face face-front"
            :class="{ 'is-cover': k === 0, 'is-clickable': k === flipped }"
            @click.stop="onFrontClick(k)"
          >
            <template v-if="k === 0">
              <div class="cover-grid"></div>
              <div class="cover-content">
                <svg class="cover-heart" viewBox="0 0 24 24" aria-hidden="true">
                  <path
                    d="M12 21.35l-1.45-1.32C5.4 15.36 2 12.28 2 8.5 2 5.42 4.42 3 7.5 3c1.74 0 3.41.81 4.5 2.09C13.09 3.81 14.76 3 16.5 3 19.58 3 22 5.42 22 8.5c0 3.78-3.4 6.86-8.55 11.54L12 21.35z"
                  />
                </svg>
                <span class="cover-text">Tap to open</span>
              </div>
            </template>
            <figure v-else class="polaroid">
              <img
                :src="leaf.frontImage"
                alt="Foto kanan"
                draggable="false"
                @error="onImgError"
              />
            </figure>
          </div>

          <!-- Sisi belakang (tampil di kiri setelah dibalik) -->
          <div
            class="face face-back"
            :class="{ 'is-clickable': k === flipped - 1 && flipped > 1 }"
            @click.stop="onBackClick(k)"
          >
            <figure class="polaroid">
              <img
                :src="leaf.backImage"
                alt="Foto kiri"
                draggable="false"
                @error="onImgError"
              />
            </figure>
          </div>
        </div>
      </div>
    </div>

    <!-- ================= HINT / TOMBOL AKHIR ================= -->
    <div class="actions">
      <button
        v-if="isLastSpread"
        key="final"
        class="final-btn"
        type="button"
        @click="finishPhase"
      >
        Sentuh untuk kejutan terakhir
      </button>
      <p v-else-if="isOpen" key="hint" class="hint">
        Ketuk halaman kanan untuk membalik halaman 📖
      </p>
    </div>
  </section>
</template>

<script setup>
import { ref, computed, onMounted, onBeforeUnmount } from "vue";

const emit = defineEmits(["phase-complete"]);

const spreadsData = ref([
  {
    leftImage: "/image/hd.jpg",
    rightImage: "/image/hd (2).jpg",
    message: "Happy Birthday Cantik💕",
  },
  {
    leftImage: "/image/fotbar (2).jpeg",
    rightImage: "/image/fotbar (3).jpeg",
    message: "As long as you're smiling. I'm happy. Stay happy, my love",
  },
  {
    leftImage: "/image/fotbar (7).jpeg",
    rightImage: "/image/fotbar (26).jpg",
    message: "I hope the world is kind to us and give us enough time to grow old together. 🥺",
  },
  {
    leftImage: "/image/fotbar (44).jpg",
    rightImage: "/image/fotbar (45).jpg",
    message: "Thank you so much for dive into my life. 💌",
  },
  {
    leftImage: "/image/fotbar (8).jpg",
    rightImage: "/image/fotbar (14).jpg",
    message: "Good things always come to me, you are my lucky charm. 🍀",
  },
  {
    leftImage: "/image/fotbar (29).jpg",
    rightImage: "/image/fotbar (30).jpg",
    message: "Im very lucky to have you in my life. You are the best girlfriend in the whole world. 🌏",
  },
  {
    leftImage: "/image/fotbar (39).jpg",
    rightImage: "/image/fotbar (40).jpg",
    message: "The moon is beautiful isn't it? 🌙",
  },
]);

const introMessage = "Ada sesuatu yang spesial untukmu… 💌";
const FLIP_LOCK_MS = 950;
const LEAVE_MS = 850;

/*
  Model buku:
  - Ada N lembaran (leaf), N = jumlah spread.
  - Leaf 0   : depan = sampul,            belakang = foto kiri spread 0
  - Leaf k>0 : depan = foto kanan spread k-1, belakang = foto kiri spread k
  - Halaman dasar: foto kanan spread terakhir
  - `flipped` = jumlah leaf yang sudah dibalik ke kiri.
      0 = tertutup, 1 = spread 0, 2 = spread 1, ... N = spread terakhir
*/
const flipped = ref(0);
const isLocked = ref(false);
const isLeaving = ref(false);
const timers = [];

const totalLeaves = computed(() => spreadsData.value.length);
const isOpen = computed(() => flipped.value > 0);
const currentSpreadIndex = computed(() => Math.max(0, flipped.value - 1));
const isLastSpread = computed(
  () => isOpen.value && flipped.value === totalLeaves.value,
);

const displayMessage = computed(() =>
  isOpen.value
    ? spreadsData.value[currentSpreadIndex.value].message
    : introMessage,
);

const lastRightImage = computed(
  () => spreadsData.value[totalLeaves.value - 1].rightImage,
);

const leaves = computed(() =>
  spreadsData.value.map((spread, k) => ({
    frontImage: k === 0 ? null : spreadsData.value[k - 1].rightImage,
    backImage: spread.leftImage,
  })),
);

// Transform tiap leaf: rotasi di engsel kiri + offset Z agar urutan tumpukan benar
function leafStyle(k) {
  const isFlipped = k < flipped.value;
  const z = isFlipped ? k + 1 : totalLeaves.value - k;
  return {
    transform: `translateZ(${z}px) rotateY(${isFlipped ? -180 : 0}deg)`,
    zIndex: z,
  };
}

function lock() {
  isLocked.value = true;
  timers.push(setTimeout(() => (isLocked.value = false), FLIP_LOCK_MS));
}

function goNext() {
  if (isLocked.value || isLeaving.value) return;
  if (flipped.value >= totalLeaves.value) return;
  flipped.value++;
  lock();
}

function goPrev() {
  if (isLocked.value || isLeaving.value) return;
  if (flipped.value <= 1) return; // tidak menutup kembali sampul
  flipped.value--;
  lock();
}

// Klik sisi depan hanya berlaku untuk leaf paling atas di tumpukan kanan
function onFrontClick(k) {
  if (k === flipped.value) goNext();
}

// Klik sisi belakang hanya berlaku untuk leaf yang paling atas di tumpukan kiri
function onBackClick(k) {
  if (k === flipped.value - 1) goPrev();
}

function finishPhase() {
  if (isLeaving.value) return;
  isLeaving.value = true;
  timers.push(setTimeout(() => emit("phase-complete"), LEAVE_MS));
}

function onImgError(e) {
  e.target.classList.add("is-broken");
}

function onKeydown(e) {
  if (e.key === "ArrowRight") goNext();
  else if (e.key === "ArrowLeft") goPrev();
}

onMounted(() => window.addEventListener("keydown", onKeydown));
onBeforeUnmount(() => {
  window.removeEventListener("keydown", onKeydown);
  timers.forEach(clearTimeout);
});
</script>

<style scoped>
.phase3 {
  --stage-w: min(100%, 780px, max(360px, calc((100dvh - 320px) * 1.6)));
  --closed-scale: 1;
  --flip-ms: 1000ms;
  --glow: 0 0 18px rgba(255, 20, 147, 0.55);

  width: 100%;
  min-height: 100dvh;
  box-sizing: border-box;
  padding: 24px 14px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 28px;
  color: var(--text-dark);
  font-family: var(--font-ui);
  overflow-x: hidden;
  overflow-x: clip;

  /* Animasi masuk: meluncur dari bawah layar ke tengah (lanjutan surat terbang) */
  animation: slideUp 1.3s cubic-bezier(0.2, 0.9, 0.25, 1) backwards;
}

.phase3.is-leaving {
  opacity: 0;
  transform: translateY(-24px);
  pointer-events: none;
  transition:
    opacity 0.8s ease,
    transform 0.8s ease;
}

/* ================= TEXTBOX ================= */
.message-box {
  width: var(--stage-w);
  box-sizing: border-box;
  min-height: 116px;
  padding: 22px 28px;
  display: flex;
  align-items: center;
  justify-content: center;
  text-align: center;
  /* DIBENAHI: Menggunakan warna murni agar kotak tidak tembus pandang/hilang */
  background: #fafafa !important;
  border: 2px solid #ff1493 !important;
  border-radius: 22px;
  box-shadow:
    0 0 18px rgba(255, 20, 147, 0.55),
    0 14px 34px rgba(0, 0, 0, 0.18);
  position: relative;
  z-index: 10;
}

.message-text {
  margin: 0;
  font-family: "Dancing Script", cursive;
  font-weight: 700;
  font-size: clamp(1.8rem, 5vw, 2.6rem);
  line-height: 1.35;
  color: #202020 !important;
  animation: fadeInText 0.8s ease forwards;
}

@keyframes fadeInText {
  from {
    opacity: 0;
    transform: translateY(8px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* ================= SCENE & BUKU ================= */
.book-scene {
  position: relative;
  width: var(--stage-w);
  perspective: 2200px;
  margin-bottom: 18px;
}

.book {
  position: relative;
  width: 100%;
  aspect-ratio: 8 / 5; /* dua halaman portrait 4:5 berdampingan */
  transform-style: preserve-3d;
  /* Tertutup: hanya setengah kanan (sampul) yang terlihat, digeser ke tengah */
  transform: translateX(calc(var(--closed-scale) * -25%))
    scale(var(--closed-scale));
  transition: transform var(--flip-ms) cubic-bezier(0.645, 0.045, 0.355, 1);
}

.book.is-open {
  transform: translateX(0) scale(1);
}

.book-board {
  position: absolute;
  inset: -10px;
  border-radius: 18px;
  background: linear-gradient(145deg, #1c0b22, #3a0a2c);
  border: 1px solid var(--neon-pink);
  box-shadow:
    var(--glow),
    0 24px 40px rgba(0, 0, 0, 0.35);
  transform: translateZ(-3px);
  opacity: 0;
  transition: opacity 0.6s ease 0.2s;
  pointer-events: none;
}

.book.is-open .book-board {
  opacity: 1;
}

.book-shadow {
  position: absolute;
  left: 50%;
  bottom: -30px;
  width: 46%;
  height: 26px;
  transform: translateX(-50%);
  background: radial-gradient(
    ellipse at center,
    rgba(255, 20, 147, 0.4),
    transparent 70%
  );
  filter: blur(8px);
  transition: width var(--flip-ms) ease;
  pointer-events: none;
}

.book-shadow.is-open {
  width: 92%;
}

/* ================= LEAF & FACE ================= */
.leaf {
  position: absolute;
  top: 0;
  left: 50%;
  width: 50%;
  height: 100%;
  transform-origin: left center; /* engsel di tulang buku */
  transform-style: preserve-3d;
  transition: transform var(--flip-ms) cubic-bezier(0.645, 0.045, 0.355, 1);
}

.face,
.base-page {
  position: absolute;
  inset: 0;
  overflow: hidden;
  background: var(--bg-polaroid);
  backface-visibility: hidden;
  -webkit-backface-visibility: hidden;
}

.base-page {
  left: 50%;
  width: 50%;
  inset: 0 auto 0 50%;
  border-radius: 0 12px 12px 0;
}

.face-front {
  border-radius: 0 12px 12px 0;
}

.face-back {
  border-radius: 12px 0 0 12px;
  transform: rotateY(180deg);
}

.face.is-clickable {
  cursor: pointer;
}

/* Bayangan lipatan di dekat tulang buku */
.face-front::after,
.base-page::after,
.face-back::after {
  content: "";
  position: absolute;
  inset: 0;
  pointer-events: none;
}

.face-front:not(.is-cover)::after,
.base-page::after {
  background: linear-gradient(to right, rgba(0, 0, 0, 0.2), transparent 14%);
}

.face-back::after {
  background: linear-gradient(to left, rgba(0, 0, 0, 0.2), transparent 14%);
}

/* ================= POLAROID ================= */
.polaroid {
  position: relative;
  width: 100%;
  height: 100%;
  margin: 0;
  box-sizing: border-box;
  padding: 5% 5% 15%;
}

.polaroid::after {
  content: "♥";
  position: absolute;
  left: 50%;
  bottom: 4.5%;
  transform: translateX(-50%);
  color: var(--neon-pink);
  font-size: 1.1rem;
  line-height: 1;
  text-shadow: 0 0 8px rgba(255, 20, 147, 0.6);
}

.polaroid img {
  display: block;
  width: 100%;
  height: 100%;
  object-fit: cover;
  background: #ececec;
  border: 1px solid #e4e4e4;
  box-sizing: border-box;
  -webkit-user-drag: none;
  user-select: none;
}

.polaroid img.is-broken {
  visibility: hidden;
}

/* ================= SAMPUL ================= */
.face-front.is-cover {
  background: linear-gradient(145deg, #1a0b1f 0%, #2d0a24 55%, #4a0a35 100%);
  box-shadow:
    inset 0 0 0 2px var(--neon-pink),
    inset 0 0 30px rgba(255, 20, 147, 0.35),
    var(--glow);
}

.face-front.is-cover::before {
  content: "";
  position: absolute;
  top: 0;
  bottom: 0;
  left: 0;
  width: 16px;
  background: linear-gradient(to right, rgba(0, 0, 0, 0.55), transparent);
  pointer-events: none;
}

.cover-grid {
  position: absolute;
  inset: 0;
  background-image:
    repeating-linear-gradient(
      0deg,
      rgba(255, 20, 147, 0.07) 0 1px,
      transparent 1px 22px
    ),
    repeating-linear-gradient(
      90deg,
      rgba(255, 20, 147, 0.07) 0 1px,
      transparent 1px 22px
    );
  pointer-events: none;
}

.cover-content {
  position: relative;
  height: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 16px;
}

.cover-heart {
  width: 34%;
  max-width: 120px;
  fill: var(--neon-pink);
  filter: drop-shadow(0 0 10px var(--neon-pink))
    drop-shadow(0 0 22px rgba(255, 20, 147, 0.6));
  animation: beat 1.6s ease-in-out infinite;
}

.cover-text {
  font-family: var(--font-ui);
  font-size: clamp(0.8rem, 2.2vw, 1.05rem);
  font-weight: 500;
  letter-spacing: 3px;
  text-transform: uppercase;
  color: #fff;
  text-shadow: 0 0 10px var(--neon-pink);
  animation: pulse 1.9s ease-in-out infinite;
}

/* ================= AKSI ================= */
.actions {
  min-height: 70px; /* Ruang dilebarkan sedikit untuk efek mengambang */
  display: flex;
  align-items: center;
  justify-content: center;
  text-align: center;
  position: relative;
  z-index: 20;
}

.hint {
  margin: 0;
  font-family: var(--font-ui);
  font-size: 0.9rem;
  color: #FAFAFA;
  opacity: 0.75;
  animation: pulse 2.2s ease-in-out infinite;
}

.final-btn {
  padding: 15px 30px;
  border: none;
  border-radius: 999px;
  /* DIBENAHI: Memaksa warna muncul tanpa perlu di-hover */
  background: linear-gradient(135deg, #FF1493, #FF69B4) !important; 
  color: #FFFFFF !important; 
  font-family: var(--font-ui);
  font-size: 1rem;
  font-weight: 600;
  letter-spacing: 0.3px;
  cursor: pointer;
  
  /* DIBENAHI: Animasi baru, mengambang dan menyala bersamaan */
  animation: floatAndGlow 2.5s ease-in-out infinite;
  
  display: block;
  opacity: 1 !important;
  visibility: visible !important;
  -webkit-tap-highlight-color: transparent;
}

.final-btn:active {
  transform: scale(0.95); /* Efek saat ditekan */
}

/* KEYFRAME BARU: Animasi Mengambang (Float) */
@keyframes floatAndGlow {
  0%, 100% {
    transform: translateY(0);
    box-shadow: 0 0 15px rgba(255, 20, 147, 0.5), 0 5px 15px rgba(0,0,0,0.2);
  }
  50% {
    /* Tombol terangkat ke atas sejauh 10px */
    transform: translateY(-10px);
    box-shadow: 0 0 30px rgba(255, 20, 147, 0.9), 0 15px 25px rgba(0,0,0,0.3);
  }
}

/* ================= KEYFRAMES ================= */
@keyframes slideUp {
  from {
    opacity: 0;
    transform: translateY(100vh);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes beat {
  0%,
  100% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.1);
  }
}

@keyframes pulse {
  0%,
  100% {
    opacity: 0.6;
  }
  50% {
    opacity: 1;
  }
}

@keyframes glowPulse {
  0%,
  100% {
    box-shadow:
      0 0 14px rgba(255, 20, 147, 0.5),
      0 10px 24px rgba(0, 0, 0, 0.25);
  }
  50% {
    box-shadow:
      0 0 30px rgba(255, 20, 147, 0.9),
      0 10px 24px rgba(0, 0, 0, 0.25);
  }
}

/* ================= RESPONSIF ================= */
@media (max-width: 640px) {
  .phase3 {
    --closed-scale: 1.2; /* sampul tertutup diperbesar agar lega di HP */
    gap: 34px;
  }

  .message-box {
    min-height: 96px;
    padding: 16px 18px;
    border-radius: 18px;
  }

  .polaroid {
    padding: 4% 4% 14%;
  }

  .polaroid::after {
    font-size: 0.8rem;
    bottom: 3.5%;
  }

  .face-front.is-cover::before {
    width: 10px;
  }

  .cover-content {
    gap: 10px;
  }

  .cover-text {
    letter-spacing: 2px;
  }

  .final-btn {
    padding: 14px 22px;
    font-size: 0.92rem;
  }
}

@media (prefers-reduced-motion: reduce) {
  .phase3 {
    --flip-ms: 400ms;
    animation-duration: 0.5s;
  }

  .cover-heart,
  .cover-text,
  .hint,
  .final-btn {
    animation: none;
  }
}
</style>
