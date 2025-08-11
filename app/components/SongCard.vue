<script setup lang="ts">
type Track = { id: string; title: string; author: string; img?: string };

const props = defineProps<{
  track: Track;
  playing: boolean;
  progress: number; // 0..100
}>();

const emit = defineEmits<{
  (e: "play"): void;
  (e: "pause"): void;
  (e: "seek", value: number): void;
  (e: "next"): void;
  (e: "prev"): void;
}>();

function onToggle() {
  props.playing ? emit("pause") : emit("play");
}
function onSeek(e: Event) {
  emit("seek", Number((e.target as HTMLInputElement).value));
}
</script>

<template>
  <div class="song-card">
    <img v-if="track.img" :src="track.img" class="cover" alt="" />

    <div class="meta">
      <h3>{{ track.title }}</h3>
      <p>{{ track.author }}</p>
    </div>

    <input
      class="progress"
      type="range"
      min="0"
      max="100"
      step="0.1"
      :value="progress"
      :style="{ '--val': progress + '%' }"
      @input="onSeek"
    />

    <div class="controls">
      <!-- Prev -->
      <button class="icon" @click="$emit('prev')" aria-label="Previous">
        <svg viewBox="0 0 24 24" width="20" height="20" fill="currentColor">
          <path d="M6 5h2v14H6zM20 6v12l-9-6 9-6z" />
        </svg>
      </button>

      <!-- Play / Pause -->
      <button
        class="icon primary"
        @click="onToggle"
        :aria-label="playing ? 'Pause' : 'Play'"
      >
        <svg
          v-if="!playing"
          viewBox="0 0 24 24"
          width="22"
          height="22"
          fill="currentColor"
        >
          <path d="M8 5v14l11-7z" />
        </svg>
        <svg
          v-else
          viewBox="0 0 24 24"
          width="22"
          height="22"
          fill="currentColor"
        >
          <path d="M6 5h4v14H6zM14 5h4v14h-4z" />
        </svg>
      </button>

      <!-- Next -->
      <button class="icon" @click="$emit('next')" aria-label="Next">
        <svg viewBox="0 0 24 24" width="20" height="20" fill="currentColor">
          <path d="M16 19h2V5h-2zM4 18V6l9 6-9 6z" />
        </svg>
      </button>
    </div>
  </div>
</template>

<style>
.song-card {
  width: 340px;
  margin: 40px auto;
  padding: 16px;
  border-radius: 16px;
  background: rgba(0, 0, 0, 0.35);
  color: #fff;
  backdrop-filter: blur(6px);
  display: grid;
  grid-template-rows: auto auto auto auto;
  gap: 12px;
}
.cover {
  width: 100%;
  aspect-ratio: 1/1;
  object-fit: cover;
  border-radius: 12px;
}

.meta {
  text-align: center;
}
.meta h3 {
  margin: 0 0 4px;
  font-size: 18px;
}
.meta p {
  margin: 0;
  opacity: 0.8;
  font-size: 13px;
}

/* --- icon buttons --- */
.controls {
  display: flex;
  justify-content: space-between;
  gap: 10px;
}
.icon {
  width: 44px;
  height: 44px;
  border-radius: 999px;
  border: 0;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  background: rgba(255, 255, 255, 0.12);
  color: #fff;
  cursor: pointer;
}
.icon.primary {
  background: #ff4d8d;
}
.icon:hover {
  filter: brightness(1.15);
}

/* --- progress bar with NO visible thumb --- */
.progress {
  --val: 0%;
  -webkit-appearance: none;
  appearance: none;
  height: 8px;
  border-radius: 999px;
  outline: none;
  background: linear-gradient(
    to right,
    #ff4d8d var(--val),
    rgba(255, 255, 255, 0.25) var(--val)
  );
}

/* WebKit (Chrome/Safari) */
.progress::-webkit-slider-runnable-track {
  height: 8px;
  border-radius: 999px;
  background: transparent;
}
.progress::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 0;
  height: 0;
  opacity: 0;
}

/* Firefox */
.progress::-moz-range-track {
  height: 8px;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.25);
}
.progress::-moz-range-progress {
  height: 8px;
  border-radius: 999px;
  background: #ff4d8d;
}
.progress::-moz-range-thumb {
  width: 0;
  height: 0;
  border: 0;
  opacity: 0;
}
</style>
