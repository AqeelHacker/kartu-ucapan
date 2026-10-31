<template>
  <div class="phase-container" :class="{ 'fade-out-phase': isPhaseEnding }">
    <div class="envelope-wrapper" @click="openEnvelope">
      <div class="envelope">
        <!-- 1. Lapisan Belakang Amplop (Jatuh ke Bawah) -->
        <div class="envelope-back" :class="{ 'drop-down': isLeaving }"></div>

        <!-- 2. Lapisan Kertas (Terbang ke Atas) -->
        <div
          class="letter"
          :class="{ 'letter-out': isOpened, 'letter-fly-up': isLeaving }"
        >
          <div class="small-heart">❤️</div>
        </div>

        <!-- 3. Lapisan Depan Amplop (Jatuh ke Bawah bersama Belakangnya) -->
        <div class="envelope-front" :class="{ 'drop-down': isLeaving }">
          <div class="envelope-wings"></div>
          <div class="flap" :class="{ 'is-opened': isOpened }"></div>
          <div class="wax-seal" :class="{ 'hide-seal': isOpened }">N</div>
        </div>
      </div>

      <p class="tap-hint" :class="{ 'fade-out': isOpened }">Tap to open</p>
    </div>
  </div>
</template>

<script setup>
import { ref } from "vue";

const emit = defineEmits(["phase-complete"]);
const isOpened = ref(false);
const isLeaving = ref(false);
const isPhaseEnding = ref(false);

const openEnvelope = () => {
  if (isOpened.value) return;

  const bgm = document.getElementById("bgm");
  if (bgm) bgm.play().catch((e) => console.log(e));

  isOpened.value = true; // Penutup terbuka, kertas naik sedikit

  // Jeda 2.5 detik untuk melihat kertas ditarik dari amplop
  setTimeout(() => {
    isLeaving.value = true; // Amplop jatuh, Kertas terbang
  }, 2500);

  // Layar perlahan pudar agar transisi ke Fase 3 sangat mulus
  setTimeout(() => {
    isPhaseEnding.value = true;
  }, 3500);

  // Pindah ke Fase 3 (Kartu Polaroid)
  setTimeout(() => {
    emit("phase-complete");
  }, 4500);
};
</script>

<style scoped>
.phase-container {
  width: 100vw;
  height: 100vh;
  position: relative;
  display: flex;
  justify-content: center;
  align-items: center;
  transition: opacity 1s ease;
  background: transparent;
  z-index: 10;
}
.fade-out-phase {
  opacity: 0;
}

.envelope-wrapper {
  position: relative;
  z-index: 10;
  cursor: pointer;
  animation: float 3s ease-in-out infinite;
  display: flex;
  flex-direction: column;
  align-items: center;
}
@keyframes float {
  0%,
  100% {
    transform: translateY(0px);
  }
  50% {
    transform: translateY(-15px);
  }
}

.envelope {
  position: relative;
  width: 300px;
  height: 200px;
}

/* 1. LAPISAN BELAKANG */
.envelope-back {
  position: absolute;
  inset: 0;
  background: #e8e8e8;
  border-radius: 5px;
  box-shadow: 0 15px 35px rgba(0, 0, 0, 0.5);
  z-index: 1;
  transition:
    transform 1.2s ease-in,
    opacity 1s ease-in;
}

/* 2. LAPISAN KERTAS */
.letter {
  position: absolute;
  top: 10px;
  left: 5%;
  width: 90%;
  height: 90%;
  background: #fff;
  border-radius: 5px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.15);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 2; /* Diapit oleh lapisan belakang dan depan amplop */
  transition: transform 1s cubic-bezier(0.2, 0.8, 0.2, 1);
}
.small-heart {
  font-size: 2.5rem;
  animation: pulse 1.5s infinite;
  opacity: 0;
  transition: opacity 0.5s ease 0.8s;
}
@keyframes pulse {
  0%,
  100% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.2);
  }
}

/* Saat kertas baru muncul setengah dari amplop */
.letter.letter-out {
  transform: translateY(-160px);
  z-index: 10;
  transition-delay: 0.4s;
}
.letter.letter-out .small-heart {
  opacity: 1;
}

/* Saat kertas terbang jauh ke atas, memisahkan diri dari amplop */
.letter.letter-fly-up {
  transform: translateY(-100vh) scale(1.2);
  opacity: 0;
  transition:
    transform 1.2s ease-in,
    opacity 1s ease-in 0.2s;
}

/* 3. LAPISAN DEPAN */
.envelope-front {
  position: absolute;
  inset: 0;
  z-index: 6;
  pointer-events: none;
  transition:
    transform 1.2s ease-in,
    opacity 1s ease-in;
}
.envelope-front > * {
  pointer-events: auto;
} /* Mengembalikan interaksi klik pada anak elemennya */
.envelope-wings {
  position: absolute;
  bottom: 0;
  left: 0;
  width: 0;
  height: 0;
  border-left: 150px solid #f0f0f0;
  border-right: 150px solid #f0f0f0;
  border-bottom: 100px solid #f8f8f8;
  border-top: 100px solid transparent;
  border-radius: 0 0 5px 5px;
}
.flap {
  position: absolute;
  top: 0;
  left: 0;
  width: 0;
  height: 0;
  border-left: 150px solid transparent;
  border-right: 150px solid transparent;
  border-top: 110px solid #e0e0e0;
  transform-origin: top;
  transition: transform 0.6s cubic-bezier(0.4, 0, 0.2, 1);
}
.flap.is-opened {
  transform: rotateX(180deg);
}
.wax-seal {
  position: absolute;
  top: 90px;
  left: 50%;
  transform: translateX(-50%);
  width: 50px;
  height: 50px;
  background: var(--neon-pink);
  border-radius: 50%;
  display: flex;
  justify-content: center;
  align-items: center;
  color: #fff;
  font-family: var(--font-romantic);
  font-size: 1.5rem;
  font-weight: bold;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.3);
  transition: opacity 0.3s ease;
}
.hide-seal {
  opacity: 0;
}

/* Animasi jatuh untuk Lapisan Depan dan Belakang secara bersamaan */
.drop-down {
  transform: translateY(100vh);
  opacity: 0;
}

.tap-hint {
  margin-top: 20px;
  font-family: var(--font-ui);
  font-size: 0.9rem;
  color: var(--text-light);
  letter-spacing: 2px;
  opacity: 0.8;
  transition: opacity 0.5s ease;
}
.fade-out {
  opacity: 0;
}
</style>
