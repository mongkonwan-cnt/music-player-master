<script setup lang="ts">
type Song = {
  id: string;
  title: string;
  author: string;
  src: string;
  img?: string;
};

const songs: Song[] = [
  {
    id: "1",
    title: "Forest Lullaby",
    author: "Artist 1",
    src: "/resources/audio/forest-lullaby-110624.mp3",
    img: "/resources/cover-1.jpg",
  },
  {
    id: "2",
    title: "Lost in city lights",
    author: "Artist 2",
    src: "/resources/audio/lost-in-city-lights-145038.mp3",
    img: "/resources/cover-2.jpg",
  },
];

const index = ref(0);
const audio = ref<HTMLAudioElement | null>(null);
const playing = ref(false);
const progress = ref(0);

const current = computed(() => songs[index.value]);

function setSource(i: number) {
  index.value = i;
  const a = audio.value;
  if (!a) return;
  a.src = songs[i].src;
  a.load();
  progress.value = 0;
}

async function playPause(forcePlay = false) {
  const a = audio.value;
  if (!a) return;
  if (forcePlay || a.paused) {
    await a.play().catch(() => {});
    playing.value = !a.paused;
  } else {
    a.pause();
    playing.value = false;
  }
}

function pause() {
  audio.value?.pause();
  playing.value = false;
}

function nextSong() {
  const was = playing.value;
  setSource((index.value + 1) % songs.length);
  if (was) playPause(true);
}

function prevSong() {
  const was = playing.value;
  setSource((index.value - 1 + songs.length) % songs.length);
  if (was) playPause(true);
}

function onTimeUpdate() {
  const a = audio.value;
  if (!a || !a.duration) return;
  progress.value = (a.currentTime / a.duration) * 100;
}

function onSeek(val: number) {
  const a = audio.value;
  if (!a || !a.duration) return;
  progress.value = val;
  a.currentTime = (val / 100) * a.duration;
}

onMounted(() => setSource(0));
</script>

<template>
  <div class="page">
    <SongCard
      :track="{
        id: current.id,
        title: current.title,
        author: current.author,
        img: current.img,
      }"
      :playing="playing"
      :progress="progress"
      @play="playPause(true)"
      @pause="pause"
      @seek="onSeek"
      @next="nextSong"
      @prev="prevSong"
    />

    <audio
      ref="audio"
      @timeupdate="onTimeUpdate"
      @ended="nextSong"
      @pause="playing = false"
      @play="playing = true"
    />
  </div>
</template>

<style>
.page {
  padding: 40px 16px;
}
</style>
