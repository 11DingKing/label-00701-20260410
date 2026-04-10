<script setup>
import { ref, watch, onMounted, nextTick } from 'vue'

const props = defineProps({
  currentTrack: Object,
  isPlaying: Boolean
})

const emit = defineEmits(['toggle-play', 'prev-track', 'next-track', 'toggle-loop', 'shuffle', 'toast'])

const audio = ref(null)
const currentTime = ref(0)
const duration = ref(0)
const progress = ref(0)
const isSeeking = ref(false)
const isLoop = ref(false)
const playbackRate = ref('1')
const switchMode = ref('end')
const fixedDuration = ref(30)

/**
 * 定时切换模式的累计播放时长
 * 
 * 【业务逻辑说明】
 * accumulatedTime 记录的是"实际播放时长"，而非"音频进度位置"。
 * 
 * 设计选择依据：
 * 1. 用户拖拽进度条（Seek）不会重置 accumulatedTime
 * 2. 这意味着"30秒切换"是指"累计播放满30秒"，而非"播放到音频的30秒位置"
 * 3. 此设计更符合"试听"场景：用户可能反复拖拽试听不同片段，但总试听时长达到阈值后自动切换
 * 4. 若需要"播放到固定位置切换"的逻辑，应改用 audio.currentTime 判断
 * 
 * 重置时机：
 * - 切换曲目时重置
 * - 切换模式时重置
 * - 曲目播放结束时重置
 */
let accumulatedTime = 0
let lastUpdateTime = 0
let currentTrackId = null

const formatTime = (s) => {
  if (!s || isNaN(s)) return '0:00'
  return Math.floor(s / 60) + ':' + String(Math.floor(s % 60)).padStart(2, '0')
}

// 加载并播放曲目
const loadTrack = (track, shouldPlay = false) => {
  if (!audio.value || !track) return
  
  console.log(`[PlayerControls] 加载曲目: ${track.name}, URL: ${track.url?.substring(0, 50)}...`)
  
  // 重置播放状态
  currentTime.value = 0
  progress.value = 0
  duration.value = track.duration || 0
  accumulatedTime = 0
  lastUpdateTime = 0
  currentTrackId = track.id
  
  // 加载新曲目
  audio.value.src = track.url
  audio.value.playbackRate = parseFloat(playbackRate.value)
  audio.value.load()
  
  if (shouldPlay) {
    audio.value.play().catch((err) => {
      console.error(`[PlayerControls] 播放失败: ${track.name}`, err)
      emit('toast', '播放失败，请重试')
    })
    lastUpdateTime = Date.now()
  }
}

// 监听曲目变化
watch(() => props.currentTrack, (newTrack, oldTrack) => {
  if (newTrack && newTrack.id !== currentTrackId) {
    console.log(`[PlayerControls] 曲目变化: ${oldTrack?.name || '无'} -> ${newTrack.name}`)
    loadTrack(newTrack, props.isPlaying)
  }
}, { deep: true })

// 监听播放状态变化
watch(() => props.isPlaying, (playing) => {
  if (!audio.value) return
  
  // 如果没有加载曲目，先加载
  if (!audio.value.src && props.currentTrack) {
    loadTrack(props.currentTrack, playing)
    return
  }
  
  if (!audio.value.src) return
  
  if (playing) {
    console.log(`[PlayerControls] 开始播放: ${props.currentTrack?.name || '未知'}`)
    audio.value.play().catch((err) => {
      console.error(`[PlayerControls] 播放失败`, err)
      emit('toast', '播放失败，请重试')
    })
    lastUpdateTime = Date.now()
  } else {
    console.log(`[PlayerControls] 暂停播放`)
    audio.value.pause()
    if (lastUpdateTime > 0) {
      accumulatedTime += (Date.now() - lastUpdateTime) / 1000
      lastUpdateTime = 0
    }
  }
})

// 组件挂载后初始化
onMounted(() => {
  nextTick(() => {
    if (props.currentTrack && audio.value) {
      console.log(`[PlayerControls] 组件挂载, 初始化曲目: ${props.currentTrack.name}`)
      loadTrack(props.currentTrack, props.isPlaying)
    }
  })
})

const updateProgress = () => {
  if (!audio.value || !audio.value.duration || isSeeking.value) return
  currentTime.value = audio.value.currentTime
  duration.value = audio.value.duration
  progress.value = (currentTime.value / duration.value) * 100
  
  // 定时切换逻辑：基于累计播放时长（非音频进度位置）
  // 详见 accumulatedTime 变量注释
  if (switchMode.value === 'time' && !isLoop.value && props.isPlaying) {
    const acc = accumulatedTime + (lastUpdateTime > 0 ? (Date.now() - lastUpdateTime) / 1000 : 0)
    if (acc >= fixedDuration.value) {
      console.log(`[PlayerControls] 定时切换触发: 累计播放 ${acc.toFixed(1)}s >= ${fixedDuration.value}s`)
      accumulatedTime = 0
      lastUpdateTime = Date.now()
      emit('next-track')
    }
  }
}

const onLoaded = () => { if (audio.value) duration.value = audio.value.duration }

const handleEnded = () => {
  console.log(`[PlayerControls] 曲目播放结束: ${props.currentTrack?.name || '未知'}, 循环模式: ${isLoop.value}`)
  if (isLoop.value) {
    audio.value.currentTime = 0
    audio.value.play()
  } else {
    accumulatedTime = 0
    emit('next-track')
  }
}

const handleError = (e) => {
  const errorCode = audio.value?.error?.code
  const errorMsg = audio.value?.error?.message || '未知错误'
  console.error(`[PlayerControls] 音频播放错误: ${props.currentTrack?.name || '未知'}, 错误码: ${errorCode}, 错误信息: ${errorMsg}`, e)
  emit('toast', '音频加载失败')
}

const getPos = (e, rect) => {
  const x = e.touches ? e.touches[0].clientX : e.clientX
  return Math.max(0, Math.min(1, (x - rect.left) / rect.width))
}

const startSeek = (e) => {
  if (!audio.value?.duration) return
  e.preventDefault()
  isSeeking.value = true
  document.body.style.userSelect = 'none'
  const rect = e.currentTarget.getBoundingClientRect()
  let seekVal = getPos(e, rect) * audio.value.duration
  currentTime.value = seekVal
  progress.value = (seekVal / audio.value.duration) * 100
  const onMove = (ev) => {
    ev.preventDefault()
    seekVal = getPos(ev, rect) * audio.value.duration
    currentTime.value = seekVal
    progress.value = (seekVal / audio.value.duration) * 100
  }
  const onEnd = () => {
    audio.value.currentTime = seekVal
    isSeeking.value = false
    document.body.style.userSelect = ''
    // 注意：Seek 操作不重置 accumulatedTime，详见变量注释中的业务逻辑说明
    console.log(`[PlayerControls] 进度拖拽完成: ${seekVal.toFixed(1)}s, 累计播放时长保持不变`)
    document.removeEventListener('mousemove', onMove)
    document.removeEventListener('mouseup', onEnd)
    document.removeEventListener('touchmove', onMove)
    document.removeEventListener('touchend', onEnd)
  }
  document.addEventListener('mousemove', onMove)
  document.addEventListener('mouseup', onEnd)
  document.addEventListener('touchmove', onMove, { passive: false })
  document.addEventListener('touchend', onEnd)
}

const toggleLoop = () => {
  isLoop.value = !isLoop.value
  console.log(`[PlayerControls] 循环模式切换: ${isLoop.value ? '开启' : '关闭'}`)
  emit('toast', isLoop.value ? '单曲循环已开启' : '单曲循环已关闭')
}

const updateRate = () => {
  if (audio.value) audio.value.playbackRate = parseFloat(playbackRate.value)
  console.log(`[PlayerControls] 播放速度调整: ${playbackRate.value}x`)
  emit('toast', '播放速度 ' + playbackRate.value + 'x')
}

const onModeChange = () => {
  accumulatedTime = 0
  if (props.isPlaying) lastUpdateTime = Date.now()
  console.log(`[PlayerControls] 切换模式变更: ${switchMode.value}, 定时时长: ${fixedDuration.value}s`)
  emit('toast', switchMode.value === 'end' ? '播放完毕后切换' : '每 ' + fixedDuration.value + ' 秒切换')
}
</script>

<template>
  <div class="now-playing">
    <div class="album-art">
      <div class="vinyl-effect" :class="{ playing: isPlaying }"></div>
      <svg class="album-art-icon" viewBox="0 0 24 24" fill="currentColor" width="64" height="64">
        <path d="M12 3v10.55c-.59-.34-1.27-.55-2-.55-2.21 0-4 1.79-4 4s1.79 4 4 4 4-1.79 4-4V7h4V3h-6z" opacity="0.4"/>
      </svg>
    </div>
    <div class="track-info">
      <h2 class="track-title">{{ currentTrack?.name || '未选择曲目' }}</h2>
      <p class="track-artist">本地音频</p>
    </div>
    <div class="progress-section">
      <div class="progress-bar-container" :class="{ seeking: isSeeking }" @mousedown="startSeek" @touchstart="startSeek">
        <div class="progress-fill" :style="{ width: progress + '%' }">
          <div class="progress-thumb" :class="{ active: isSeeking }"></div>
        </div>
      </div>
      <div class="time-info">
        <span>{{ formatTime(currentTime) }}</span>
        <span>{{ formatTime(duration) }}</span>
      </div>
    </div>
    <div class="controls">
      <button class="control-btn" :class="{ active: isLoop }" @click="toggleLoop">
        <svg viewBox="0 0 24 24" fill="currentColor"><path d="M7 7h10v3l4-4-4-4v3H5v6h2V7zm10 10H7v-3l-4 4 4 4v-3h12v-6h-2v4z"/></svg>
      </button>
      <button class="control-btn" @click="$emit('prev-track')">
        <svg viewBox="0 0 24 24" fill="currentColor"><path d="M6 6h2v12H6zm3.5 6l8.5 6V6z"/></svg>
      </button>
      <button class="control-btn play-btn" @click="$emit('toggle-play')">
        <svg v-if="!isPlaying" viewBox="0 0 24 24" fill="currentColor"><path d="M8 5v14l11-7z"/></svg>
        <svg v-else viewBox="0 0 24 24" fill="currentColor"><path d="M6 19h4V5H6v14zm8-14v14h4V5h-4z"/></svg>
      </button>
      <button class="control-btn" @click="$emit('next-track')">
        <svg viewBox="0 0 24 24" fill="currentColor"><path d="M6 18l8.5-6L6 6v12zM16 6v12h2V6h-2z"/></svg>
      </button>
      <button class="control-btn" @click="$emit('shuffle')">
        <svg viewBox="0 0 24 24" fill="currentColor"><path d="M10.59 9.17L5.41 4 4 5.41l5.17 5.17 1.42-1.41zM14.5 4l2.04 2.04L4 18.59 5.41 20 17.96 7.46 20 9.5V4h-5.5zm.33 9.41l-1.41 1.41 3.13 3.13L14.5 20H20v-5.5l-2.04 2.04-3.13-3.13z"/></svg>
      </button>
    </div>
    <div class="extra-controls">
      <div class="control-chip">
        <select v-model="playbackRate" @change="updateRate">
          <option value="0.5">0.5x</option>
          <option value="0.75">0.75x</option>
          <option value="1">1.0x</option>
          <option value="1.25">1.25x</option>
          <option value="1.5">1.5x</option>
          <option value="2">2.0x</option>
        </select>
      </div>
      <div class="control-chip">
        <select v-model="switchMode" @change="onModeChange">
          <option value="end">播完切换</option>
          <option value="time">定时切换</option>
        </select>
      </div>
      <div class="control-chip time-chip" v-if="switchMode === 'time'">
        <input type="number" v-model.number="fixedDuration" min="1" max="3600" />
        <span class="unit">秒</span>
      </div>
    </div>
    <audio ref="audio" @timeupdate="updateProgress" @ended="handleEnded" @loadedmetadata="onLoaded" @error="handleError"></audio>
  </div>
</template>
