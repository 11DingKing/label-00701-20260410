<script setup>
const props = defineProps({
  playlist: {
    type: Array,
    required: true
  },
  currentIndex: {
    type: Number,
    default: 0
  },
  isPlaying: {
    type: Boolean,
    default: false
  }
})

const emit = defineEmits(['play-track', 'shuffle'])

const formatTime = (seconds) => {
  if (!seconds || isNaN(seconds)) return '--:--'
  const mins = Math.floor(seconds / 60)
  const secs = Math.floor(seconds % 60)
  return `${mins}:${secs.toString().padStart(2, '0')}`
}
</script>

<template>
  <div class="playlist-section">
    <div class="playlist-header">
      <h2 class="playlist-title">
        播放列表
        <span class="playlist-count">{{ playlist.length }} 首曲目</span>
      </h2>
      <button class="shuffle-btn" @click="$emit('shuffle')">
        <svg viewBox="0 0 24 24" fill="currentColor">
          <path d="M10.59 9.17L5.41 4 4 5.41l5.17 5.17 1.42-1.41zM14.5 4l2.04 2.04L4 18.59 5.41 20 17.96 7.46 20 9.5V4h-5.5zm.33 9.41l-1.41 1.41 3.13 3.13L14.5 20H20v-5.5l-2.04 2.04-3.13-3.13z"/>
        </svg>
        随机排列
      </button>
    </div>

    <div class="playlist-table">
      <div class="playlist-table-header">
        <span>#</span>
        <span>标题</span>
        <span class="header-duration">
          <svg viewBox="0 0 24 24" fill="currentColor" width="16" height="16">
            <path d="M11.99 2C6.47 2 2 6.48 2 12s4.47 10 9.99 10C17.52 22 22 17.52 22 12S17.52 2 11.99 2zM12 20c-4.42 0-8-3.58-8-8s3.58-8 8-8 8 3.58 8 8-3.58 8-8 8zm.5-13H11v6l5.25 3.15.75-1.23-4.5-2.67z"/>
          </svg>
        </span>
      </div>
      <div class="playlist-items">
        <div
          v-for="(track, index) in playlist"
          :key="track.id"
          class="playlist-item"
          :class="{ active: currentIndex === index }"
          @click="$emit('play-track', index)"
        >
          <div>
            <span class="item-number">{{ index + 1 }}</span>
            <span class="item-play-icon">
              <svg v-if="currentIndex === index && isPlaying" width="14" height="14" viewBox="0 0 24 24" fill="currentColor">
                <rect x="6" y="4" width="4" height="16" rx="1"/>
                <rect x="14" y="4" width="4" height="16" rx="1"/>
              </svg>
              <svg v-else width="14" height="14" viewBox="0 0 24 24" fill="currentColor">
                <path d="M8 5v14l11-7z"/>
              </svg>
            </span>
          </div>
          <div class="item-info">
            <div class="item-cover">
              <svg viewBox="0 0 24 24" fill="currentColor" width="20" height="20" style="opacity: 0.4">
                <path d="M12 3v10.55c-.59-.34-1.27-.55-2-.55-2.21 0-4 1.79-4 4s1.79 4 4 4 4-1.79 4-4V7h4V3h-6z"/>
              </svg>
            </div>
            <span class="item-title">{{ track.name }}</span>
          </div>
          <span class="item-duration">{{ formatTime(track.duration) }}</span>
        </div>
      </div>
    </div>
  </div>
</template>
