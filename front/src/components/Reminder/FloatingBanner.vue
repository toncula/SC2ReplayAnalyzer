<template>
  <div 
    ref="containerRef"
    class="banner-container" 
    :style="containerStyle"
    data-tauri-drag-region
  >
    <!-- 内容区域 -->
    <div v-html="text || '示例提示文字'"></div>

    <!-- [新增] 交互式播放控制 -->
    <div v-if="player && player.enabled" class="floating-controls">
      <div class="progress-bar-wrapper">
        <!-- 进度条滑块 -->
        <input 
          type="range" 
          class="mini-slider" 
          min="0" 
          :max="player.maxTime" 
          step="1"
          :value="player.currentTime"
          @input="onSeek"
          @mousedown.stop 
        />
      </div>
      <div class="controls-row">
        <!-- 播放/暂停按钮 -->
        <button class="mini-btn" @click.stop="togglePlay">
          {{ player.isPlaying ? '⏸' : '▶️' }}
        </button>
        <!-- 时间显示 -->
        <span class="mini-time">{{ formatTime(player.currentTime) }}</span>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, nextTick } from 'vue'
import { listen, emit } from '@tauri-apps/api/event'
import { getCurrentWindow, LogicalSize } from '@tauri-apps/api/window'

// 显示文字 & 样式
const text = ref('')
const containerStyle = ref<Record<string, any>>({})
const containerRef = ref<HTMLElement | null>(null)
const player = ref<any>(null) // 存储播放器状态

let unlisten: (() => void) | null = null
let resizeObserver: ResizeObserver | null = null

function formatTime(seconds: number) {
  const m = Math.floor(seconds / 60)
  const s = Math.floor(seconds % 60)
  return `${m}:${s.toString().padStart(2, '0')}`
}

function onSeek(e: Event) {
  const target = e.target as HTMLInputElement
  const val = parseInt(target.value)
  // 发送 seek 请求给主窗口
  emit('screen-banner:control', { type: 'seek', value: val })
}

function togglePlay() {
  emit('screen-banner:control', { type: 'toggle' })
}

const updateWindowSize = async () => {
  await nextTick()
  const el = containerRef.value
  if (!el) return

  // 获取实际渲染宽度
  const width = el.offsetWidth + 2
  const height = el.offsetHeight + 2

  const appWindow = getCurrentWindow()
  await appWindow.setSize(new LogicalSize(width, height))
}

onMounted(async () => {
  const appWindow = getCurrentWindow()

  unlisten = await listen('screen-banner:update', async (event) => {
    const payload = event.payload as { 
      text: string; 
      style: Record<string, any>; 
      behavior: Record<string, any>;
      locked: boolean;
    }
    
    text.value = payload.text
    // 获取播放器状态
    if (payload.behavior && payload.behavior.player) {
      player.value = payload.behavior.player
    } else {
      player.value = null
    }
    
    containerStyle.value = {
      ...payload.style, // 继承主窗口发来的基础样式

      width: 'fit-content', 
      height: 'fit-content',
      maxWidth: '1200px',
      margin: '0',
      
      // 强制 Block 布局以支持图文混排 + 底部控件栏
      display: 'block', 
      alignItems: 'unset',
      justifyContent: 'unset',

      whiteSpace: 'pre-wrap', 
      
      // 关键：如果锁定，悬浮窗整体鼠标穿透。
      // 这意味着用户将无法点击进度条，这是系统限制。
      // 用户必须解锁才能操作进度条。
      pointerEvents: payload.locked ? 'none' : 'auto'
    }
    
    await appWindow.setIgnoreCursorEvents(payload.locked)
    
    if (!payload.locked) {
      await appWindow.setFocus()
    }
  })

  if (containerRef.value) {
    resizeObserver = new ResizeObserver(() => {
      updateWindowSize()
    })
    resizeObserver.observe(containerRef.value)
  }

  await emit('screen-banner:ready')
})

onBeforeUnmount(() => {
  unlisten && unlisten()
  if (resizeObserver) resizeObserver.disconnect()
})
</script>

<style>
:global(body) {
  margin: 0;
  padding: 0;
  background: transparent !important; 
  overflow: hidden;
}

.banner-container {
  width: fit-content;
  text-align: left; 
  border-radius: 10px;
  user-select: none; 
  cursor: default;
  min-width: 0;
  position: relative; 
}

/* 播放器控件样式 */
.floating-controls {
  margin-top: 8px;
  padding-top: 4px;
  border-top: 1px solid rgba(255, 255, 255, 0.1);
  display: flex;
  flex-direction: column;
  gap: 4px;
  /* 尝试在解锁状态下允许交互 */
  pointer-events: auto; 
}

.progress-bar-wrapper {
  width: 100%;
  height: 10px;
  display: flex;
  align-items: center;
}

.mini-slider {
  -webkit-appearance: none;
  width: 100%;
  height: 4px;
  background: rgba(255, 255, 255, 0.2);
  border-radius: 2px;
  outline: none;
  cursor: pointer;
}
.mini-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 12px;
  height: 12px;
  border-radius: 50%;
  background: #3b82f6;
  cursor: pointer;
  transition: transform 0.1s;
}
.mini-slider::-webkit-slider-thumb:hover {
  transform: scale(1.2);
}

.controls-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 12px;
  color: rgba(255, 255, 255, 0.8);
}

.mini-btn {
  background: none;
  border: none;
  color: inherit;
  cursor: pointer;
  font-size: 16px;
  padding: 0 4px;
}
.mini-btn:hover {
  color: white;
  transform: scale(1.1);
}
</style>