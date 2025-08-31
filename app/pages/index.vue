<script setup lang="ts">
import { ref, computed, onMounted, onBeforeUnmount } from "vue";

type Song = {
  id: string;
  title: string;
  author: string;
  src: string;
  img?: string;
  category?: string;
};

const songs: Song[] = [
  {
    id: "1",
    title: "Forest Lullaby",
    author: "Artist forest",
    src: "/resources/audio/forest-lullaby-110624.mp3",
    img: "/resources/cover-1.jpg",
    category: "normal",
  },
  {
    id: "2",
    title: "Lost in city lights",
    author: "Artist lofi",
    src: "/resources/audio/lost-in-city-lights-145038.mp3",
    img: "/resources/cover-2.jpg",
    category: "lofi",
  },
  {
    id: "3",
    title: "Lost series",
    author: "Artist lofi",
    src: "/resources/audio/lost-in-city-lights-145038.mp3",
    img: "/resources/cover-2.jpg",
    category: "lofi",
  },
  {
    id: "4",
    title: "Rock star",
    author: "Artist rock",
    src: "/resources/audio/lost-in-city-lights-145038.mp3",
    img: "/resources/cover-2.jpg",
    category: "rock",
  },
  {
    id: "5",
    title: "Other buttom mind",
    author: "Artist rock",
    src: "/resources/audio/lost-in-city-lights-145038.mp3",
    img: "/resources/cover-2.jpg",
    category: "",
  },
];

const CATEGORY_ALL = "ALL";

const categoryFilter = ref<string>(CATEGORY_ALL);
const index = ref(0);
const audio = ref<HTMLAudioElement | null>(null);
const playing = ref(false);
const progress = ref(0);
const time = ref(0);
const duration = ref(0);

const timeText = computed(() => formatTime(time.value));
const durationText = computed(() =>
  duration.value ? formatTime(duration.value) : "--:--"
);
const current = computed(() => songs[index.value]);

/* ---- Utils / playback ---- */
const hasAudio = () => !!audio.value;
const clamp = (v: number, min: number, max: number) =>
  Math.min(Math.max(v, min), max);
const wrap = (n: number, len: number) => (n + len) % len;

function formatTime(s: number) {
  if (!isFinite(s) || s < 0) return "0:00";
  const h = Math.floor(s / 3600);
  const m = Math.floor((s % 3600) / 60);
  const sec = Math.floor(s % 60)
    .toString()
    .padStart(2, "0");
  return h > 0 ? `${h}:${m.toString().padStart(2, "0")}:${sec}` : `${m}:${sec}`;
}

function setSource(i: number) {
  index.value = i;
  if (!hasAudio()) return;
  const a = audio.value!;
  a.src = songs[i].src;
  a.load();
  progress.value = 0;
  duration.value = 0;
  updateMediaMetadata();
}

async function playPause(forcePlay = false) {
  if (!hasAudio()) return;
  const a = audio.value!;
  if (forcePlay || a.paused) {
    await a.play().catch(() => {});
    playing.value = !a.paused;
  } else {
    a.pause();
    playing.value = false;
  }
  updatePlaybackState();
}
function pause() {
  if (!hasAudio()) return;
  audio.value!.pause();
  playing.value = false;
  updatePlaybackState();
}
function nextSong() {
  const was = playing.value;
  setSource(wrap(index.value + 1, songs.length));
  if (was) playPause(true);
}
function prevSong() {
  const was = playing.value;
  setSource(wrap(index.value - 1, songs.length));
  if (was) playPause(true);
}

function onTimeUpdate() {
  const a = audio.value;
  if (!a || !a.duration) return;
  time.value = a.currentTime;
  duration.value = a.duration;
  progress.value = (a.currentTime / a.duration) * 100;
  // @ts-ignore
  navigator.mediaSession?.setPositionState?.({
    duration: a.duration,
    position: a.currentTime,
    playbackRate: a.playbackRate || 1,
  });
}
function onLoadedMetadata() {
  if (audio.value) duration.value = audio.value.duration || 0;
}
function onSeek(percent: number) {
  const a = audio.value;
  if (!a || !a.duration) return;
  progress.value = percent;
  a.currentTime = (percent / 100) * a.duration;
  time.value = a.currentTime;
}
function seekBy(seconds: number) {
  const a = audio.value;
  if (!a || !a.duration) return;
  a.currentTime = clamp(a.currentTime + seconds, 0, a.duration);
}

/* ---------- Keyboard ---------- */
function onKey(e: KeyboardEvent) {
  const t = e.target as HTMLElement | null;
  const tag = t?.tagName?.toLowerCase();
  if (tag === 'input' || tag === 'textarea' || t?.isContentEditable) return;

  switch (e.code) {
    case 'MediaPlayPause':
    case 'Space':
    case 'KeyK':
      e.preventDefault(); playPause(); break;
    case 'MediaTrackNext':
      e.preventDefault(); nextSong(); break;
    case 'MediaTrackPrevious':
      e.preventDefault(); prevSong(); break;
    case 'KeyL':
    case 'ArrowRight':
      seekBy(10); break;
    case 'KeyJ':
    case 'ArrowLeft':
      seekBy(-10); break;
  }
}

/* ---------- Media Session ---------- */
function initMediaSessionHandlers() {
  // @ts-ignore
  const ms = navigator.mediaSession;
  if (!ms) return;
  ms.setActionHandler('play', () => playPause(true));
  ms.setActionHandler('pause', () => pause());
  ms.setActionHandler('nexttrack', () => nextSong());
  ms.setActionHandler('previoustrack', () => prevSong());
  ms.setActionHandler('seekforward', (e: any) => seekBy(e?.seekOffset ?? 10));
  ms.setActionHandler('seekbackward', (e: any) => seekBy(-(e?.seekOffset ?? 10)));
  ms.setActionHandler('seekto', (e: any) => {
    if (audio.value && e?.seekTime != null) audio.value.currentTime = e.seekTime;
  });
}

function updateMediaMetadata() {
  // @ts-ignore
  const ms = navigator.mediaSession;
  if (!ms || !current.value) return;

  const art = current.value.img
    ? [{ src: new URL(current.value.img, location.origin).toString(), sizes: '512x512', type: 'image/jpeg' }]
    : [];

  // @ts-ignore
  ms.metadata = new MediaMetadata({
    title: current.value.title,
    artist: current.value.author,
    artwork: art,
  });
  updatePlaybackState();
}

function updatePlaybackState() {
  // @ts-ignore
  const ms = navigator.mediaSession;
  if (!ms) return;
  ms.playbackState = playing.value ? 'playing' : 'paused';
}

/* ---------- UI helpers ---------- */
function selectSong(i: number) {
  const was = playing.value;
  setSource(i);
  if (was) playPause(true);
}

onMounted(() => {
  if (songs.length) setSource(0);
  initMediaSessionHandlers();
  window.addEventListener("keydown", onKey);
});
onBeforeUnmount(() => {
  window.removeEventListener("keydown", onKey);
});
</script>

<template>
  <div class="page">
    <div class="player">
      <!-- LEFT -->

      <!-- plz coomment this to test new version -->
      <SongList
        :songs="songs"
        :selectedIndex="index"
        v-model:categoryFilter="categoryFilter"
        @select="selectSong"
      />

      <!-- <SongListV2/> -->

      <!-- RIGHT: player card -->
      <div class="song-card-wrap">
        <SongCard
          :track="{
            id: current.id,
            title: current.title,
            author: current.author,
            img: current.img,
          }"
          :playing="playing"
          :progress="progress"
          :timeText="timeText"
          :durationText="durationText"
          @play="playPause(true)"
          @pause="pause"
          @seek="onSeek"
          @next="nextSong"
          @prev="prevSong"
        />
      </div>

      <audio
        ref="audio"
        @loadedmetadata="onLoadedMetadata"
        @timeupdate="onTimeUpdate"
        @ended="nextSong"
        @pause="
          () => {
            playing.value = false;
            updatePlaybackState();
          }
        "
        @play="
          () => {
            playing.value = true;
            updatePlaybackState();
          }
        "
      />
    </div>
  </div>
</template>

<style>
.page {
  min-height: 100svh;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 24px;
}
.player {
  display: grid;
  grid-template-columns: clamp(260px, 30vw, 380px) minmax(360px, 1fr);
  gap: 24px;
  align-items: center;
  max-width: 1200px;
  width: 100%;
}
.player > * {
  min-width: 0;
}
.song-card-wrap {
  padding-left: 100px;
  min-width: 0;
}
@media (max-width: 900px) {
  .player {
    grid-template-columns: 2fr;
  }
  .song-card-wrap {
    order: 1;
  }
}
</style>
