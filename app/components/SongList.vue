<script setup lang="ts">
import { computed } from "vue";

type Song = {
  id: string;
  title: string;
  author: string;
  src: string;
  img?: string;
  category?: string;
};

const CATEGORY_ALL = "ALL";
const CATEGORY_OTHER = "Other";

const props = defineProps<{
  songs: Song[];
  selectedIndex: number;
  categoryFilter: string;
}>();

const emit = defineEmits<{
  (e: "update:categoryFilter", v: string): void;
  (e: "select", index: number): void;
}>();

const normCat = (c?: string) => c || CATEGORY_OTHER;

const categories = computed(() => {
  const uniq = Array.from(new Set(props.songs.map((s) => normCat(s.category))));
  return [CATEGORY_ALL, ...uniq];
});

const visible = computed(() =>
  props.songs
    .map((s, idx) => ({ s, idx }))
    .filter(
      ({ s }) =>
        props.categoryFilter === CATEGORY_ALL ||
        normCat(s.category) === props.categoryFilter
    )
);

function selectFilter(item: string) {
  emit("update:categoryFilter", item);
}
function isActiveFilter(item: string) {
  return props.categoryFilter === item;
}
function chipClass(item: string) {
  return ["chip", { active: isActiveFilter(item) }];
}
function filterLabel(item: string) {
  return item.toUpperCase();
}
</script>

<template>
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
        :class="['song-row', { active: idx === selectedIndex }]"
      >
        <button
          class="song-item"
          @click="$emit('select', idx)"
          :aria-current="idx === selectedIndex"
        >
          <img v-if="s.img" :src="s.img" alt="" class="thumb" />
          <div class="meta">
            <div class="title">{{ s.title }}</div>
            <div class="author">{{ s.author }}</div>
          </div>
        </button>
      </div>
    </div>
  </div>
</template>

<style scoped>
:root {
  --grad-a: #0ea5e9;
  --grad-b: #7c3aed;
}
.song-list-card {
  border-radius: 16px;
  overflow: hidden;
  background: linear-gradient(
    180deg,
    rgba(14, 165, 233, 0.1),
    rgba(124, 58, 237, 0.1)
  );
  border: 1px solid rgba(0, 0, 0, 0.1);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  box-shadow: 0 10px 30px rgba(2, 6, 23, 0.12);
  display: flex;
  flex-direction: column;
  max-height: 60vh;
}
.list-header {
  position: sticky;
  top: 0;
  z-index: 1;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  padding: 12px;
  background: linear-gradient(135deg, var(--grad-a), var(--grad-b));
  color: #fff;
}
.chip-group {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}
.chip {
  padding: 6px 12px;
  border-radius: 999px;
  border: 1px solid rgba(255, 255, 255, 0.55);
  background: transparent;
  color: #fff;
  font-weight: 700;
  letter-spacing: 0.06em;
  opacity: 0.95;
  cursor: pointer;
}
.chip:hover {
  background: rgba(255, 255, 255, 0.18);
}
.chip.active {
  background: #fff;
  color: #111827;
  border-color: transparent;
}
.list-count {
  font-size: 0.75rem;
  padding: 4px 10px;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.18);
}
.list-body {
  overflow: auto;
  padding: 10px;
  flex: 1;
}
.song-row + .song-row {
  margin-top: 8px;
}
.song-item {
  display: grid;
  grid-template-columns: 48px 1fr;
  column-gap: 12px;
  align-items: center;
  width: 100%;
  padding: 10px;
  border-radius: 12px;
  border: 1px solid rgba(0, 0, 0, 0.08);
  background: rgba(255, 255, 255, 0.4);
  cursor: pointer;
  text-align: left;
}
.song-item:hover {
  background: linear-gradient(
    135deg,
    rgba(14, 165, 233, 0.12),
    rgba(124, 58, 237, 0.12)
  );
}
.song-row.active .song-item {
  background: linear-gradient(
    135deg,
    rgba(14, 165, 233, 0.22),
    rgba(124, 58, 237, 0.22)
  );
  border-color: transparent;
  box-shadow: 0 0 0 2px rgba(124, 58, 237, 0.25) inset;
}
.thumb {
  width: 48px;
  height: 48px;
  object-fit: cover;
  border-radius: 8px;
}
.meta {
  min-width: 0;
  display: grid;
  grid-template-rows: calc(1.25em * 2) 1.1em;
}
.meta .title {
  display: -webkit-box;
  -webkit-line-clamp: 2;
  overflow: hidden;
  line-height: 1.25;
  font-weight: 700;
}
.meta .author {
  display: -webkit-box;
  -webkit-line-clamp: 2;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  opacity: 0.75;
  line-height: 1.1;
}
</style>
