<script setup>
import { ref, computed, onUnmounted } from 'vue'
import FileUploader from './components/FileUploader.vue'
import PlayerControls from './components/PlayerControls.vue'
import Playlist from './components/Playlist.vue'
import Toast from './components/Toast.vue'

// 状态
const playlist = ref([])
const currentIndex = ref(0)
const isPlaying = ref(false)
const toast = ref({ show: false, message: '' })

// 组件引用
const fileUploader = ref(null)
const playerControls = ref(null)

let toastTimer = null

// 计算属性
const currentTrack = computed(() => {
  if (playlist.value.length === 0) return null
  return playlist.value[currentIndex.value] || null
})

// Toast 提示
const showToast = (message) => {
  toast.value = { show: true, message }
  if (toastTimer) clearTimeout(toastTimer)
  toastTimer = setTimeout(() => {
    toast.value.show = false
  }, 2000)
}

// 文件处理
const onFilesAdded = (tracks) => {
  const isFirstAdd = playlist.value.length === 0
  console.log(`[App] 接收到 ${tracks.length} 个音频文件, 当前列表: ${playlist.value.length} 首`)
  playlist.value.push(...tracks)
  
  // 导入后自动随机排列（符合 Prompt 要求：导入 -> 随后随机排列 -> 播放）
  if (playlist.value.length >= 2) {
    // Fisher-Yates 洗牌算法
    for (let i = playlist.value.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1))
      ;[playlist.value[i], playlist.value[j]] = [playlist.value[j], playlist.value[i]]
    }
    console.log(`[App] 播放列表已随机排列, 共 ${playlist.value.length} 首`)
    showToast(`已添加 ${tracks.length} 首曲目并随机排列`)
  } else {
    showToast(`已添加 ${tracks.length} 首曲目`)
  }
  
  // 如果是第一次添加，自动选中第一首
  if (isFirstAdd && tracks.length > 0) {
    currentIndex.value = 0
    console.log(`[App] 首次添加, 自动选中第一首: ${playlist.value[0]?.name}`)
  }
}

const onFileError = (errors) => {
  console.warn(`[App] 文件处理错误:`, errors)
  errors.forEach(err => showToast(err))
}

// 播放控制
const togglePlay = () => {
  if (playlist.value.length === 0) {
    console.log(`[App] 播放列表为空, 无法播放`)
    showToast('请先添加音频文件')
    return
  }
  
  const newState = !isPlaying.value
  isPlaying.value = newState
  console.log(`[App] 播放状态切换: ${newState ? '播放' : '暂停'}`)
  showToast(newState ? '正在播放' : '已暂停')
}

const playTrack = (index) => {
  const isSameTrack = currentIndex.value === index
  currentIndex.value = index
  
  if (isSameTrack && isPlaying.value) {
    // 点击当前正在播放的曲目，暂停
    isPlaying.value = false
    console.log(`[App] 点击当前曲目, 暂停播放`)
    showToast('已暂停')
  } else {
    isPlaying.value = true
    console.log(`[App] 播放曲目 [${index}]: ${playlist.value[index].name}`)
    showToast(`正在播放: ${playlist.value[index].name}`)
  }
}

const prevTrack = () => {
  if (playlist.value.length === 0) return
  const oldIndex = currentIndex.value
  currentIndex.value = currentIndex.value > 0 
    ? currentIndex.value - 1 
    : playlist.value.length - 1
  console.log(`[App] 上一曲: [${oldIndex}] -> [${currentIndex.value}] ${playlist.value[currentIndex.value].name}`)
  isPlaying.value = true
}

const nextTrack = () => {
  if (playlist.value.length === 0) return
  const oldIndex = currentIndex.value
  currentIndex.value = currentIndex.value < playlist.value.length - 1 
    ? currentIndex.value + 1 
    : 0
  console.log(`[App] 下一曲: [${oldIndex}] -> [${currentIndex.value}] ${playlist.value[currentIndex.value].name}`)
  isPlaying.value = true
}

// 随机排列
const shufflePlaylist = () => {
  if (playlist.value.length < 2) {
    console.log(`[App] 曲目不足, 无法随机排列`)
    showToast('至少需要 2 首曲目')
    return
  }
  
  const currentTrackData = playlist.value[currentIndex.value]
  
  // Fisher-Yates 洗牌算法
  for (let i = playlist.value.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1))
    ;[playlist.value[i], playlist.value[j]] = [playlist.value[j], playlist.value[i]]
  }
  
  // 保持当前播放曲目的索引
  if (currentTrackData) {
    currentIndex.value = playlist.value.findIndex(t => t.id === currentTrackData.id)
  }
  
  console.log(`[App] 手动随机排列完成, 当前曲目索引: ${currentIndex.value}`)
  showToast('已随机排列')
}

// 循环状态变化
const onLoopChange = (isLoop) => {
  // 可以在这里处理循环状态变化的逻辑
}

// 清理
onUnmounted(() => {
  console.log(`[App] 组件卸载, 清理 ${playlist.value.length} 个 Blob URL`)
  if (toastTimer) clearTimeout(toastTimer)
  playlist.value.forEach(track => URL.revokeObjectURL(track.url))
})
</script>

<template>
  <div class="app">
    <header class="header">
      <div class="header-content">
        <div class="logo">
          <div class="logo-icon">
            <svg viewBox="0 0 24 24" fill="currentColor" width="22" height="22">
              <path d="M12 3v10.55c-.59-.34-1.27-.55-2-.55-2.21 0-4 1.79-4 4s1.79 4 4 4 4-1.79 4-4V7h4V3h-6z"/>
            </svg>
          </div>
          <span>AudioFlow</span>
        </div>
      </div>
    </header>

    <main class="main-content">
      <!-- 上传区域（无曲目时显示） -->
      <section class="upload-section" v-if="playlist.length === 0">
        <FileUploader 
          ref="fileUploader"
          @files-added="onFilesAdded"
          @error="onFileError"
        />
      </section>

      <!-- 播放器区域 -->
      <section class="player-section" v-if="playlist.length > 0">
        <PlayerControls
          ref="playerControls"
          :current-track="currentTrack"
          :playlist="playlist"
          :is-playing="isPlaying"
          @toggle-play="togglePlay"
          @prev-track="prevTrack"
          @next-track="nextTrack"
          @toggle-loop="onLoopChange"
          @shuffle="shufflePlaylist"
          @toast="showToast"
        />

        <Playlist
          :playlist="playlist"
          :current-index="currentIndex"
          :is-playing="isPlaying"
          @play-track="playTrack"
          @shuffle="shufflePlaylist"
        />
      </section>

      <!-- 添加更多（有曲目时显示） -->
      <div v-if="playlist.length > 0" class="add-more-section">
        <FileUploader
          ref="fileUploader"
          class="upload-area-mini"
          @files-added="onFilesAdded"
          @error="onFileError"
        />
      </div>
    </main>

    <Toast :show="toast.show" :message="toast.message" />
  </div>
</template>
