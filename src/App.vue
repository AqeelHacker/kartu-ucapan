<template>
  <main>
    <audio
      id="bgm"
      ref="audioPlayer"
      preload="auto"
      @timeupdate="handleTimeUpdate"
      @ended="handleEnded"
    ></audio>

    <!-- Bintang tampil di Fase 0 (layar sandi / buket) dan Fase 2-4. Fase 1 (Matrix) tanpa bintang -->
    <div v-if="currentPhase !== 1" class="global-decorations">
      <div class="dense-stars"></div>

      <div class="floating-emojis">
        <div
          v-for="(item, index) in activeEmojis"
          :key="item.id"
          class="emoji-item"
          :style="{
            left: item.x + '%',
            animationDuration: item.speed + 's',
            animationDelay: item.delay + 's',
          }"
          @animationend="resetEmoji(index)"
        >
          {{ item.char }}
        </div>
      </div>

      <div v-if="showDisk" class="music-player">
        <div class="disk-wrapper" @click="spawnNoteOnClick">
          <div class="disk"><div class="disk-center"></div></div>

          <div class="notes-container">
            <span
              v-for="note in activeNotes"
              :key="note.id"
              class="music-note"
              :class="note.direction"
              :style="{ transform: `translateX(${note.offset}px)` }"
              @animationend="removeNote(note.id)"
            >
              🎵
            </span>
          </div>
        </div>
      </div>
    </div>

    <!-- PANGGUNG FASE -->
    <LandingView
      v-if="currentPhase === 0"
      v-model:unlocked="landingUnlocked"
      @play-music="playCustomAudio"
      @stop-music="stopMusic"
      @start-surprise="currentPhase = 1"
    />
    <Phase1Matrix v-if="currentPhase === 1" @phase-complete="nextPhase" />
    <Phase2Envelope v-if="currentPhase === 2" @phase-complete="nextPhase" />
    <Phase3Polaroid v-if="currentPhase === 3" @phase-complete="nextPhase" />
    <Phase4Finale v-if="currentPhase === 4" @back-to-menu="currentPhase = 0" />
  </main>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, watch } from "vue";
import LandingView from "./components/landing/LandingView.vue";
import Phase1Matrix from "./components/surprise/Phase1Matrix.vue";
import Phase2Envelope from "./components/surprise/Phase2Envelope.vue";
import Phase3Polaroid from "./components/surprise/Phase3Polaroid.vue";
import Phase4Finale from "./components/surprise/Phase4Finale.vue";

// 0 = Halaman Awal (sandi, buket, surat, kado), 1-4 = Fase Kejutan
const currentPhase = ref(0);
const nextPhase = () => {
  currentPhase.value++;
};

// Disimpan di sini (bukan di LandingView) supaya tetap true saat kembali dari Fase 4
const landingUnlocked = ref(false);

// Piringan hitam: tampil di Fase 2-4, dan di Fase 0 setelah sandi benar
const showDisk = computed(
  () => currentPhase.value >= 2 || (currentPhase.value === 0 && landingUnlocked.value)
);

// ================= AUDIO SETTINGS (EDIT DI SINI) =================
// Helper waktu: mmss(menit, detik) -> detik. Contoh: mmss(1, 30) = 1 menit 30 detik
const mmss = (menit, detik = 0) => menit * 60 + detik;

const TRACKS = {
  // Lagu Halaman Awal (setelah sandi benar, sampai kado dibuka)
  landing: {
    src: "/audio/lagu (2).mp3",
    start: mmss(0, 0), // mulai dari menit:detik berapa
    end: mmss(3, 48), // selesai di menit:detik berapa (null = sampai lagu habis)
    volume: 0.5, // 0 - 1
  },
  // Lagu Fase Kejutan (mulai saat masuk Fase 2, setelah Matrix)
  surprise: {
    src: "/audio/lagu.mp3", // GANTI dengan nama file lagu surprise-mu
    start: mmss(2, 13),
    end: mmss(3, 50),
    end: null,
    volume: 0.5,
  },
};
// =================================================================

const audioPlayer = ref(null);
let currentTrack = null; // 'landing' | 'surprise' (track yang sedang dimuat di elemen audio)

// Fase 0 -> lagu landing, Fase 2-4 -> lagu surprise
const trackNameForPhase = () => (currentPhase.value === 0 ? "landing" : "surprise");

// Muat track ke elemen audio (hanya kalau berbeda dari yang sedang dimuat)
const loadTrack = (name) => {
  const audio = audioPlayer.value;
  const track = TRACKS[name];
  if (!audio || !track) return null;

  if (currentTrack !== name) {
    currentTrack = name;
    audio.src = track.src;
    audio.load();
    // Lompat ke detik mulai begitu metadata lagu siap
    audio.addEventListener(
      "loadedmetadata",
      () => {
        audio.currentTime = track.start;
      },
      { once: true }
    );
  }
  return { audio, track };
};

// Loop di titik akhir yang ditentukan
const handleTimeUpdate = (event) => {
  const track = TRACKS[currentTrack];
  if (!track || !track.end) return;
  const audio = event.target;
  if (audio.currentTime >= track.end) {
    audio.currentTime = track.start;
    audio.play().catch(() => {});
  }
};

// Kalau lagu habis secara alami (end: null), ulang dari titik mulai
const handleEnded = () => {
  const track = TRACKS[currentTrack];
  const audio = audioPlayer.value;
  if (!track || !audio) return;
  audio.currentTime = track.start;
  audio.play().catch(() => {});
};

// Memutar lagu sesuai fase saat ini
const playCustomAudio = async () => {
  const loaded = loadTrack(trackNameForPhase());
  if (!loaded) return;
  const { audio, track } = loaded;

  audio.volume = track.volume;
  if (audio.currentTime < track.start) {
    audio.currentTime = track.start;
  }
  try {
    await audio.play();
  } catch (err) {
    console.log("Menunggu interaksi user...");
  }
};

const stopMusic = () => {
  if (audioPlayer.value) {
    audioPlayer.value.pause();
    audioPlayer.value.currentTime = 0;
  }
  clearInterval(noteInterval);
  activeEmojis.value = [];
};

// --- LOGIKA HIASAN GLOBAL ---
const emojiList = ["❤️", "💖", "💕", "💓", "💘", "💝", "✨", "🌸"];
const activeEmojis = ref([]);
let emojiIdCounter = 0;
const activeNotes = ref([]);
let noteIdCounter = 0;
let noteInterval;

const createRandomEmoji = (withDelay = false) => {
  return {
    id: emojiIdCounter++,
    char: emojiList[Math.floor(Math.random() * emojiList.length)],
    x: Math.random() * 80 + 10,
    speed: Math.random() * 3 + 5,
    delay: withDelay ? Math.random() * 3 : 0,
  };
};

const resetEmoji = (index) => {
  activeEmojis.value[index] = createRandomEmoji(false);
};

const spawnNoteOnClick = () => {
  playCustomAudio();
  const directions = ["left", "center", "right"];
  directions.forEach((dir) => {
    activeNotes.value.push({
      id: noteIdCounter++,
      direction: dir,
      offset: (Math.random() - 0.5) * 30,
    });
  });
};

const removeNote = (id) => {
  activeNotes.value = activeNotes.value.filter((n) => n.id !== id);
};

watch(currentPhase, (newPhase) => {
  // Masuk Fase 2: mulai lagu surprise + efek hiasan
  if (newPhase === 2) {
    playCustomAudio();
    for (let i = 0; i < 4; i++) {
      activeEmojis.value.push(createRandomEmoji(true));
    }
    noteInterval = setInterval(() => {
      spawnNoteOnClick();
    }, 4000);
  }

  // Masuk Fase 1 (Matrix): lagu dan efek dimatikan seperti sebelumnya
  if (newPhase === 1) {
    stopMusic();
  }

  // Kembali ke Fase 0 (tombol Back dari Fase 4): hentikan efek Fase 2-4.
  // Karena sandi sudah terbuka, lagu landing dilanjutkan; kalau belum, pastikan hening.
  if (newPhase === 0) {
    clearInterval(noteInterval);
    activeEmojis.value = [];
    if (landingUnlocked.value) {
      playCustomAudio();
    } else {
      stopMusic();
    }
  }
});

// Preload lagu landing supaya langsung siap saat sandi benar
onMounted(() => {
  loadTrack("landing");
});

onUnmounted(() => {
  clearInterval(noteInterval);
});
</script>

<style scoped>
.global-decorations {
  position: absolute;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  pointer-events: none;
  overflow: hidden;
}
.dense-stars {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: var(--bg-space);
}
.dense-stars::after {
  content: "";
  position: absolute;
  width: 2px;
  height: 2px;
  background: transparent;
  box-shadow:
    5vw 5vh #fff,
    15vw 8vh #fff,
    25vw 3vh #fff,
    35vw 12vh #fff,
    45vw 6vh #fff,
    55vw 18vh #fff,
    65vw 4vh #fff,
    75vw 15vh #fff,
    85vw 8vh #fff,
    95vw 12vh #fff,
    10vw 25vh #fff,
    20vw 22vh #fff,
    30vw 35vh #fff,
    40vw 28vh #fff,
    50vw 33vh #fff,
    60vw 25vh #fff,
    70vw 38vh #fff,
    80vw 24vh #fff,
    90vw 32vh #fff,
    98vw 27vh #fff,
    3vw 45vh #fff,
    12vw 55vh #fff,
    22vw 48vh #fff,
    32vw 58vh #fff,
    42vw 42vh #fff,
    52vw 52vh #fff,
    62vw 45vh #fff,
    72vw 55vh #fff,
    82vw 42vh #fff,
    92vw 52vh #fff,
    8vw 65vh #fff,
    18vw 72vh #fff,
    28vw 68vh #fff,
    38vw 75vh #fff,
    48vw 63vh #fff,
    58vw 78vh #fff,
    68vw 65vh #fff,
    78vw 72vh #fff,
    88vw 62vh #fff,
    97vw 75vh #fff,
    5vw 85vh #fff,
    15vw 95vh #fff,
    25vw 82vh #fff,
    35vw 92vh #fff,
    45vw 85vh #fff,
    55vw 98vh #fff,
    65vw 82vh #fff,
    75vw 95vh #fff,
    85vw 88vh #fff,
    95vw 92vh #fff,
    4vw 15vh #fff,
    14vw 35vh #fff,
    24vw 15vh #fff,
    34vw 45vh #fff,
    44vw 25vh #fff,
    54vw 40vh #fff,
    64vw 20vh #fff,
    74vw 45vh #fff,
    84vw 18vh #fff,
    94vw 38vh #fff;
  border-radius: 50%;
  animation: twinkle 3s infinite alternate;
}
@keyframes twinkle {
  0% {
    opacity: 0.2;
  }
  100% {
    opacity: 1;
  }
}

.floating-emojis {
  position: absolute;
  width: 100%;
  height: 100%;
}
.emoji-item {
  position: absolute;
  bottom: -50px;
  font-size: 2rem;
  opacity: 0.7;
  animation-name: flyUp;
  animation-timing-function: linear;
  animation-fill-mode: forwards;
}
@keyframes flyUp {
  0% {
    transform: translateY(0) scale(0.8);
    opacity: 0;
  }
  10% {
    opacity: 0.8;
  }
  90% {
    opacity: 0.8;
  }
  100% {
    transform: translateY(-110vh) scale(1.2);
    opacity: 0;
  }
}

.music-player {
  position: absolute;
  bottom: 30px;
  right: 30px;
  pointer-events: auto;
  z-index: 50;
}
.disk-wrapper {
  position: relative;
  display: flex;
  justify-content: center;
  align-items: center;
  cursor: pointer;
}
.disk {
  width: 60px;
  height: 60px;
  background: linear-gradient(135deg, #111, #444, #111);
  border-radius: 50%;
  border: 3px solid #333;
  display: flex;
  justify-content: center;
  align-items: center;
  animation: spin 3s linear infinite;
  box-shadow: 0 0 15px var(--neon-pink-glow);
}
.disk-center {
  width: 15px;
  height: 15px;
  background: var(--neon-pink);
  border-radius: 50%;
  border: 2px solid #fff;
}
@keyframes spin {
  100% {
    transform: rotate(360deg);
  }
}

.notes-container {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  display: flex;
  justify-content: center;
  align-items: center;
}
.music-note {
  position: absolute;
  font-size: 1.5rem;
  color: var(--soft-pink);
  opacity: 0;
}
.music-note.left {
  animation: floatLeft 2.3s ease-out forwards;
}
.music-note.center {
  animation: floatCenter 1.8s ease-out forwards;
}
.music-note.right {
  animation: floatRight 1.2s ease-out forwards;
}

@keyframes floatLeft {
  0% {
    transform: translate(0, 0) scale(0.5);
    opacity: 1;
  }
  100% {
    transform: translate(-60px, -80px) scale(1.2) rotate(-30deg);
    opacity: 0;
  }
}
@keyframes floatCenter {
  0% {
    transform: translate(0, 0) scale(0.5);
    opacity: 1;
  }
  100% {
    transform: translate(0px, -100px) scale(1.2) rotate(10deg);
    opacity: 0;
  }
}
@keyframes floatRight {
  0% {
    transform: translate(0, 0) scale(0.5);
    opacity: 1;
  }
  100% {
    transform: translate(60px, -80px) scale(1.2) rotate(45deg);
    opacity: 0;
  }
}
</style>