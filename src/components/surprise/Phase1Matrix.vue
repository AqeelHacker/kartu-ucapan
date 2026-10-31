<template>
  <div class="phase-container" :class="{ 'fade-out': isFadingOut }">
    <!-- Tag Audio SFX -->
    <audio id="sfx-beep" src="/audio/beep.mp3" preload="auto"></audio>
    <audio id="sfx-pop" src="/audio/pop.mp3" preload="auto"></audio>

    <canvas ref="matrixCanvas" class="matrix-canvas"></canvas>
    
    <div class="overlay">
      <h1 v-if="countdown > 0" class="countdown-text">{{ countdown }}</h1>
      <h1 v-else :key="stepIndex" :class="isFullSentence ? 'greeting-full' : 'greeting-word'">
        {{ displayedText }}
      </h1>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';

const emit = defineEmits(['phase-complete']);
const matrixCanvas = ref(null);
const countdown = ref(3); 
const displayedText = ref(''); 
const stepIndex = ref(0); 
const isFullSentence = ref(false); 
const isFadingOut = ref(false);
let intervalId;

// Fungsi pembantu untuk memutar SFX
const playSound = (id) => {
  const audio = document.getElementById(id);
  if (audio) {
    audio.currentTime = 0; // Reset audio ke awal agar bisa diputar cepat berulang
    audio.play().catch(e => console.log(e));
  }
};

const startMatrix = () => { /* LOGIKA CANVAS SAMA SEPERTI SEBELUMNYA */
  const canvas = matrixCanvas.value;
  const ctx = canvas.getContext('2d');
  canvas.width = window.innerWidth; canvas.height = window.innerHeight;
  const letters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789@#$%^&*';
  const fontSize = 16; const columns = canvas.width / fontSize;
  const drops = []; for (let x = 0; x < columns; x++) drops[x] = 1;
  const draw = () => {
    ctx.fillStyle = 'rgba(11, 12, 16, 0.05)'; ctx.fillRect(0, 0, canvas.width, canvas.height);
    ctx.fillStyle = '#FF1493'; ctx.font = fontSize + 'px VT323';
    for (let i = 0; i < drops.length; i++) {
      const text = letters.charAt(Math.floor(Math.random() * letters.length));
      ctx.fillText(text, i * fontSize, drops[i] * fontSize);
      if (drops[i] * fontSize > canvas.height && Math.random() > 0.975) drops[i] = 0;
      drops[i]++;
    }
  };
  intervalId = setInterval(draw, 33);
};

const runSequence = () => {
  // Beep pertama saat angka 3 muncul
  playSound('sfx-beep');

  setTimeout(() => { countdown.value = 2; playSound('sfx-beep'); }, 1000); 
  setTimeout(() => { countdown.value = 1; playSound('sfx-beep'); }, 2000); 
  
  setTimeout(() => { countdown.value = 0; displayedText.value = 'HAPPY'; stepIndex.value++; playSound('sfx-pop'); }, 3000); 
  setTimeout(() => { displayedText.value = 'BIRTHDAY'; stepIndex.value++; playSound('sfx-pop'); }, 4200);
  setTimeout(() => { displayedText.value = 'TO'; stepIndex.value++; playSound('sfx-pop'); }, 5400);
  setTimeout(() => { displayedText.value = 'NAURA'; stepIndex.value++; playSound('sfx-pop'); }, 6600);
  
  setTimeout(() => { 
    displayedText.value = 'HAPPY BIRTHDAY TO NAURA'; 
    isFullSentence.value = true; 
    stepIndex.value++; 
    playSound('sfx-pop'); 
  }, 7800);

  setTimeout(() => { isFadingOut.value = true; }, 11000); 
  setTimeout(() => { emit('phase-complete'); }, 12000); 
};

onMounted(() => { startMatrix(); runSequence(); });
onUnmounted(() => { clearInterval(intervalId); });
</script>

<style scoped>
.phase-container {
  width: 100vw;
  height: 100vh;
  position: relative;
  background-color: var(--bg-space);
  transition: opacity 1s ease-in-out; 
}

.fade-out {
  opacity: 0;
}

.matrix-canvas {
  display: block;
  width: 100%;
  height: 100%;
}

.overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 10;
}

.countdown-text {
  font-family: var(--font-matrix);
  font-size: 20rem;
  color: var(--text-light);
  text-shadow: 0 0 20px var(--neon-pink);
}

/* Desain untuk kata yang muncul satu-satu (Ukuran Raksasa) */
.greeting-word {
  font-family: var(--font-matrix);
  font-size: 15rem;
  color: var(--neon-pink);
  text-shadow: 0 0 25px var(--neon-pink-glow);
  text-align: center;
  animation: popEffect 0.8s cubic-bezier(0.175, 0.885, 0.32, 1.275) forwards; /* Efek pantulan */
}

/* Desain untuk kalimat utuh (Ukuran Disesuaikan) */
.greeting-full {
  font-family: var(--font-matrix);
  font-size: 10rem;
  color: var(--neon-pink);
  text-shadow: 0 0 15px var(--neon-pink-glow);
  text-align: center;
  animation: popEffect 0.8s ease-out forwards;
}

/* Keyframes untuk efek Pop-Up membesar dari kecil */
@keyframes popEffect {
  0% {
    transform: scale(0.2);
    opacity: 0;
  }
  80% {
    transform: scale(1.1);
    opacity: 1;
  }
  100% {
    transform: scale(1);
    opacity: 1;
  }
}

/* Responsivitas Layar HP */
@media (max-width: 768px) {
  .countdown-text { font-size: 10rem; }
  .greeting-word { font-size: 5rem; }
  .greeting-full { font-size: 3rem; padding: 0 20px; }
}
</style>