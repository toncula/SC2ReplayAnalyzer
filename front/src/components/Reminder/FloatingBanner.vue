<template>
  <div 
    ref="containerRef"
    class="banner-container" 
    :style="containerStyle"
    data-tauri-drag-region
  >
    <!-- 内容区域 -->
    <div v-html="text || '示例提示文字'"></div>

    <!-- 交互式播放控制 -->
    <div v-if="player && player.enabled" class="floating-controls">
      <div class="progress-bar-wrapper">
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
        <div style="display:flex; align-items:center; gap:4px;">
          <!-- 播放/暂停 -->
          <button class="mini-btn" @click.stop="togglePlay" :title="player.isPlaying ? '暂停' : '播放'">
            {{ player.isPlaying ? '⏸' : '▶️' }}
          </button>
          
          <!-- [新增] 锁定/解锁按钮 -->
          <!-- 只有在解锁状态(isLocked=false)下，悬浮窗才可拖动。锁定后鼠标穿透(无法拖动)，但控制条区域我们设为可点击 -->
          <button 
            class="mini-btn" 
            @click.stop="toggleLock" 
            :title="isLocked ? '当前位置已锁定 (点击解锁)' : '当前可拖动 (点击锁定)'"
            :style="{ opacity: isLocked ? '0.6' : '1', color: isLocked ? '#9ca3af' : '#3b82f6' }"
          >
            {{ isLocked ? '🔒' : '🔓' }}
          </button>
        </div>

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

// 状态
const text = ref('')
const containerStyle = ref<Record<string, any>>({})
const containerRef = ref<HTMLElement | null>(null)
const player = ref<any>(null)
const isLocked = ref(true) // [新增] 本地存储锁定状态用于UI显示

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
  emit('screen-banner:control', { type: 'seek', value: val })
}

function togglePlay() {
  emit('screen-banner:control', { type: 'toggle' })
}

// [新增] 发送锁定切换请求
function toggleLock() {
  emit('screen-banner:control', { type: 'toggle-lock' })
}

const updateWindowSize = async () => {
  await nextTick()
  const el = containerRef.value
  if (!el) return
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
    isLocked.value = payload.locked // 更新锁定状态
    
    if (payload.behavior && payload.behavior.player) {
      player.value = payload.behavior.player
    } else {
      player.value = null
    }
    
    // 如果播放器开启，必须允许鼠标捕获(setIgnoreCursorEvents false)才能操作按钮
    // 如果播放器关闭，则完全遵循 locked 状态
    const playerEnabled = player.value && player.value.enabled
    const shouldIgnoreMouse = playerEnabled ? false : payload.locked

    containerStyle.value = {
      ...payload.style,
      width: 'fit-content', 
      height: 'fit-content',
      maxWidth: '1200px',
      margin: '0',
      display: 'block', 
      alignItems: 'unset',
      justifyContent: 'unset',
      whiteSpace: 'pre-wrap', 
      
      // 样式层面的穿透控制：
      // Locked: 文本区域穿透 (none)，但在 CSS 中我们会强制 controls 区域 auto
      // Unlocked: 整体可点 (auto)
      pointerEvents: payload.locked ? 'none' : 'auto'
    }
    
    await appWindow.setIgnoreCursorEvents(shouldIgnoreMouse)
    
    if (!shouldIgnoreMouse && !payload.locked) {
      await appWindow.setFocus()
    }
  })

  if (containerRef.value) {
    resizeObserver = new ResizeObserver(() => updateWindowSize())
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

.floating-controls {
  margin-top: 8px;
  padding-top: 4px;
  border-top: 1px solid rgba(255, 255, 255, 0.1);
  display: flex;
  flex-direction: column;
  gap: 4px;
  /* [关键] 强制控件区域可点击，即使父容器是 none */
  pointer-events: auto !important; 
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
  width: 10px;
  height: 10px;
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
  transition: opacity 0.2s;
}
.mini-btn:hover {
  color: white;
  transform: scale(1.1);
}
</style>