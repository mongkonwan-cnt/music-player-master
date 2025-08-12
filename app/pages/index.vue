<script setup lang="ts">
import { ref, computed, onMounted, onBeforeUnmount } from 'vue'

type Song = {
  id: string;
  title: string;
  author: string;
  src: string;
  img?: string;
  category?: string;
};

const songs: Song[] = [
  { id: "1", title: "Forest Lullaby", author: "Artist forest", src: "/resources/audio/forest-lullaby-110624.mp3", img: "/resources/cover-1.jpg", category: "normal" },
  { id: "2", title: "Lost in city lights", author: "Artist lofi",  src: "/resources/audio/lost-in-city-lights-145038.mp3", img: "/resources/cover-2.jpg", category: "lofi" },
  { id: "3", title: "Lost series", author: "Artist lofi",  src: "/resources/audio/lost-in-city-lights-145038.mp3", img: "/resources/cover-2.jpg", category: "lofi" },
  { id: "4", title: "Rock star", author: "Artist rock",  src: "/resources/audio/lost-in-city-lights-145038.mp3", img: "/resources/cover-2.jpg", category: "rock" },
  { id: "5", title: "Other buttom mind", author: "Artist rock",  src: "/resources/audio/lost-in-city-lights-145038.mp3", img: "/resources/cover-2.jpg", category: "" },
];

const CATEGORY_ALL = 'ALL';
const CATEGORY_OTHER = 'Other';

const categoryFilter = ref<string>(CATEGORY_ALL);

const index = ref(0);
const audio = ref<HTMLAudioElement | null>(null);
const playing = ref(false);
const progress = ref(0);     // 0..100
const time = ref(0);
const duration = ref(0);

const timeText = computed(() => formatTime(time.value));
const durationText = computed(() => duration.value ? formatTime(duration.value) : '--:--');
const current = computed(() => songs[index.value]);

/* ---------- Utils ---------- */
const hasAudio = () => !!audio.value;
const clamp = (v: number, min: number, max: number) => Math.min(Math.max(v, min), max);
const wrap = (n: number, len: number) => (n + len) % len;
const normCat = (c?: string) => (c || CATEGORY_OTHER);

/* ---------- Filters / List ---------- */
const categories = computed(() => {
  const uniq = Array.from(new Set(songs.map(s => normCat(s.category))));
  return [CATEGORY_ALL, ...uniq];
});

// IMPORTANT: keep original index with each visible row to avoid mismatch when filtered
const visible = computed(() =>
  songs
    .map((s, idx) => ({ s, idx }))
    .filter(({ s }) => categoryFilter.value === CATEGORY_ALL || normCat(s.category) === categoryFilter.value)
);

function selectFilter(item: string) { categoryFilter.value = item; }
function isActiveFilter(item: string) { return categoryFilter.value === item; }
function chipClass(item: string) { return ['chip', { active: isActiveFilter(item) }]; }
function filterLabel(item: string) { return item.toUpperCase(); }

/* ---------- Time format ---------- */
function formatTime(s: number) {
  if (!isFinite(s) || s < 0) return '0:00';
  const h = Math.floor(s / 3600);
  const m = Math.floor((s % 3600) / 60);
  const sec = Math.floor(s % 60).toString().padStart(2, '0');
  return h > 0 ? `${h}:${m.toString().padStart(2,'0')}:${sec}` : `${m}:${sec}`;
}

/* ---------- Source / playback ---------- */
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

/* ---------- Seek / progress ---------- */
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

/* ---------- Lifecycle ---------- */
onMounted(() => {
  if (songs.length) setSource(0);
  initMediaSessionHandlers();
  window.addEventListener('keydown', onKey);
});

onBeforeUnmount(() => {
  window.removeEventListener('keydown', onKey);
});
</script>

<template>
  <div class="page">
    <div class="player">
      <!-- Left: list card -->
      <div class="song-list-card">
        <div class="list-header">
          <div class="chip-group" role="tablist" aria-label="Song filters">
            <button
              v-for="item in categories"
              :key="item"
              :class="chipClass(item)"
              role="tab"
              :aria-selected="isActiveFilter(item)"
              @click="selectFilter(item)"
            >
              {{ filterLabel(item) }}
            </button>
          </div>
          <span class="list-count">{{ visible.length }}</span>
        </div>

        <div class="list-body">
          <div
            v-for="{ s, idx } in visible"
            :key="s.id"
            :class="['song-row', { active: idx === index }]"
          >
            <button class="song-item" @click="selectSong(idx)" :aria-current="idx === index">
              <img v-if="s.img" :src="s.img" alt="" class="thumb" />
              <div class="meta">
                <div class="title">{{ s.title }}</div>
                <div class="author">{{ s.author }}</div>
              </div>
            </button>
          </div>
        </div>
      </div>

      <!-- Right: player card -->
      <SongCard
        :track="{ id: current.id, title: current.title, author: current.author, img: current.img }"
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

      <audio
        ref="audio"
        @loadedmetadata="onLoadedMetadata"
        @timeupdate="onTimeUpdate"
        @ended="nextSong"
        @pause="() => { playing.value = false; updatePlaybackState(); }"
        @play="() => { playing.value = true; updatePlaybackState(); }"
      />
    </div>
  </div>
</template>

<style>
.page{
  min-height: 100svh;
  display: flex;
  align-items: center;
  justify-content: center;

  padding: 0px 0px 0px 500px;
}

/* 2-column layout: list | card */
.player{
  display: grid;
  grid-template-columns: 460px minmax(320px, 520px);
  gap: 24px;
  align-items: start;
}

/* song list styles */
:root{
  --grad-a: #0ea5e9; /* sky-500 */
  --grad-b: #7c3aed; /* violet-600 */
}

.song-list-card{
  width: 360px;
  border-radius: 16px;
  overflow: hidden;
  background: linear-gradient(180deg, rgba(14,165,233,.10), rgba(124,58,237,.10));
  border: 1px solid rgba(0,0,0,.10);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  box-shadow: 0 10px 30px rgba(2,6,23,.12);
}

/* header with gradient bar + chips */
.list-header{
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  padding: 12px;
  background: linear-gradient(135deg, var(--grad-a), var(--grad-b));
  color: #fff;
}

.chip-group{ display: flex; gap: 8px; flex-wrap: wrap; }
.chip{
  padding: 6px 12px;
  border-radius: 999px;
  border: 1px solid rgba(255,255,255,.55);
  background: transparent;
  color: #fff;
  font-weight: 700;
  letter-spacing: .06em;
  opacity: .95;
  cursor: pointer;
}
.chip:hover{ background: rgba(255,255,255,.18); }
.chip.active{
  background: #fff;
  color: #111827;
  border-color: transparent;
}

.list-count{
  font-size: .75rem;
  padding: 4px 10px;
  border-radius: 999px;
  background: rgba(255,255,255,.18);
}

/* body / rows */
.list-body{
  max-height: 420px;
  overflow: auto;
  padding: 10px;
}
.song-row + .song-row{ margin-top: 8px; }

.song-item{
  width: 100%;
  display: flex;
  gap: 12px;
  padding: 10px;
  border-radius: 12px;
  border: 1px solid rgba(0,0,0,.08);
  background: rgba(255,255,255,.40);
  cursor: pointer;
  text-align: left;
}
.song-item:hover{
  background: linear-gradient(135deg, rgba(14,165,233,.12), rgba(124,58,237,.12));
}
.song-row.active .song-item{
  background: linear-gradient(135deg, rgba(14,165,233,.22), rgba(124,58,237,.22));
  border-color: transparent;
  box-shadow: 0 0 0 2px rgba(124,58,237,.25) inset;
}

.thumb{ width: 48px; height: 48px; object-fit: cover; border-radius: 8px; }
.meta .title{ font-weight: 700; line-height: 1.2; }
.meta .author{ opacity: .75; font-size: .9rem; }
</style>
