<script setup>
import { ref } from 'vue'

const emit = defineEmits(['files-added', 'error'])
const fileInput = ref(null)
const isDragging = ref(false)

const SUPPORTED_FORMATS = ['audio/mpeg', 'audio/wav', 'audio/ogg', 'audio/flac', 'audio/aac', 'audio/mp4', 'audio/webm']
const SUPPORTED_EXTENSIONS = ['.mp3', '.wav', '.ogg', '.flac', '.aac', '.m4a', '.webm']

const isAudioFile = (file) => {
  // 检查 MIME 类型
  if (file.type && SUPPORTED_FORMATS.some(format => file.type.startsWith(format.split('/')[0]))) {
    console.log(`[FileUploader] 文件类型检测通过: ${file.name}, MIME: ${file.type}`)
    return true
  }
  // 检查文件扩展名作为备选
  const fileName = file.name.toLowerCase()
  const isValid = SUPPORTED_EXTENSIONS.some(ext => fileName.endsWith(ext))
  console.log(`[FileUploader] 文件扩展名检测: ${file.name}, 结果: ${isValid ? '通过' : '不支持'}`)
  return isValid
}

const validateAndLoadAudio = (file) => {
  return new Promise((resolve, reject) => {
    const url = URL.createObjectURL(file)
    const tempAudio = new Audio()
    const startTime = Date.now()
    let timeoutId = null
    
    console.log(`[FileUploader] 开始加载音频: ${file.name}, 大小: ${(file.size / 1024 / 1024).toFixed(2)}MB`)
    
    const cleanup = () => {
      if (timeoutId) clearTimeout(timeoutId)
      tempAudio.removeEventListener('loadedmetadata', onLoad)
      tempAudio.removeEventListener('error', onError)
      tempAudio.src = ''
    }
    
    const onLoad = () => {
      const loadTime = Date.now() - startTime
      const duration = tempAudio.duration
      console.log(`[FileUploader] 音频加载成功: ${file.name}, 时长: ${duration.toFixed(2)}s, 加载耗时: ${loadTime}ms`)
      cleanup()
      // 注意：不在这里释放 URL，保留给播放器使用
      resolve({
        id: `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
        name: file.name.replace(/\.[^/.]+$/, ''),
        file,
        url,
        duration
      })
    }
    
    const onError = (e) => {
      const errorCode = tempAudio.error?.code
      const errorMsg = tempAudio.error?.message || '未知错误'
      console.error(`[FileUploader] 音频加载失败: ${file.name}, 错误码: ${errorCode}, 错误信息: ${errorMsg}`, e)
      cleanup()
      URL.revokeObjectURL(url)
      reject(new Error(`无法加载音频文件: ${file.name}`))
    }
    
    tempAudio.addEventListener('loadedmetadata', onLoad)
    tempAudio.addEventListener('error', onError)
    
    // 设置超时，防止无限等待
    timeoutId = setTimeout(() => {
      console.error(`[FileUploader] 音频加载超时: ${file.name}, 已等待 10000ms`)
      cleanup()
      URL.revokeObjectURL(url)
      reject(new Error(`加载超时: ${file.name}`))
    }, 10000)
    
    tempAudio.src = url
  })
}

const handleFiles = (e) => {
  addFiles(e.target.files)
  // 重置 input，允许重复选择同一文件
  e.target.value = ''
}

const handleDrop = (e) => {
  isDragging.value = false
  addFiles(e.dataTransfer.files)
}

const addFiles = async (files) => {
  console.log(`[FileUploader] 开始处理 ${files.length} 个文件`)
  const validFiles = []
  const errors = []
  
  // 转换为数组，确保可以正确遍历
  const fileArray = Array.from(files)
  
  for (let i = 0; i < fileArray.length; i++) {
    const file = fileArray[i]
    console.log(`[FileUploader] 处理第 ${i + 1}/${fileArray.length} 个文件: ${file.name}`)
    
    if (!isAudioFile(file)) {
      const errMsg = `不支持的文件格式: ${file.name}`
      console.warn(`[FileUploader] ${errMsg}, MIME: ${file.type}`)
      errors.push(errMsg)
      continue
    }
    
    try {
      const track = await validateAndLoadAudio(file)
      validFiles.push(track)
    } catch (err) {
      console.error(`[FileUploader] 文件处理异常: ${file.name}`, err)
      errors.push(err.message)
    }
  }
  
  console.log(`[FileUploader] 处理完成: 成功 ${validFiles.length} 个, 失败 ${errors.length} 个`)
  
  if (validFiles.length > 0) {
    emit('files-added', validFiles)
  }
  
  if (errors.length > 0) {
    emit('error', errors)
  }
}

const openFilePicker = () => {
  fileInput.value.click()
}

defineExpose({ openFilePicker })
</script>

<template>
  <div
    class="upload-area"
    :class="{ dragging: isDragging }"
    @click="openFilePicker"
    @dragover.prevent="isDragging = true"
    @dragleave="isDragging = false"
    @drop.prevent="handleDrop"
  >
    <input 
      type="file" 
      ref="fileInput" 
      :multiple="true"
      accept="audio/*,.mp3,.wav,.ogg,.flac,.aac,.m4a,.webm" 
      @change="handleFiles" 
    />
    <div class="upload-icon-wrapper">
      <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
        <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4" />
        <polyline points="17 8 12 3 7 8" />
        <line x1="12" y1="3" x2="12" y2="15" />
      </svg>
    </div>
    <h2 class="upload-title">拖放音频文件到这里</h2>
    <p class="upload-subtitle">或点击选择文件</p>
    <div class="upload-formats">
      <span class="format-tag">MP3</span>
      <span class="format-tag">WAV</span>
      <span class="format-tag">FLAC</span>
      <span class="format-tag">OGG</span>
      <span class="format-tag">AAC</span>
    </div>
  </div>
</template>
