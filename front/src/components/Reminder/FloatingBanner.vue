<template>
  <div 
    ref="containerRef"
    class="banner-container" 
    :style="containerStyle"
    data-tauri-drag-region
    v-html="text || '示例提示文字'"
  >
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

let unlisten: (() => void) | null = null
let resizeObserver: ResizeObserver | null = null

// 窗口尺寸自适应
const updateWindowSize = async () => {
  await nextTick()
  const el = containerRef.value
  if (!el) return

  // 获取实际渲染宽度
  // +2 防止某些 DPI 下边缘裁切
  const width = el.offsetWidth + 2
  const height = el.offsetHeight + 2

  const appWindow = getCurrentWindow()
  await appWindow.setSize(new LogicalSize(width, height))
}

onMounted(async () => {
  const appWindow = getCurrentWindow()

  // 监听主窗口发来的更新事件
  unlisten = await listen('screen-banner:update', async (event) => {
    const payload = event.payload as { 
      text: string; 
      style: Record<string, any>; 
      behavior: Record<string, any>;
      locked: boolean;
    }
    
    text.value = payload.text
    
    // 合并样式
    containerStyle.value = {
      ...payload.style, // 继承主窗口发来的基础样式 (字体、颜色、背景等)

      // --- 悬浮窗强制样式覆盖 ---
      
      width: 'fit-content', 
      height: 'fit-content',
      maxWidth: '1200px',
      margin: '0',
      
      // 1. 布局修复：
      // 主窗口为了垂直居中使用了 flex，但在悬浮窗中，
      // 我们需要 block 布局让“进度条 div”自动换行到文字下方。
      display: 'block', 
      alignItems: 'unset',
      justifyContent: 'unset',

      // 2. 文本换行修复：
      // 保证长文本和 HTML 内容能正确换行
      whiteSpace: 'pre-wrap', 
      
      // 3. 交互设置：
      pointerEvents: payload.locked ? 'none' : 'auto'
    }
    
    // 处理鼠标穿透状态
    await appWindow.setIgnoreCursorEvents(payload.locked)
    
    // 如果解锁了，聚焦窗口以便接收键盘事件（如果有的话）
    if (!payload.locked) {
      await appWindow.setFocus()
    }
  })

  // 监听内容尺寸变化，自动调整窗口大小
  if (containerRef.value) {
    resizeObserver = new ResizeObserver(() => {
      updateWindowSize()
    })
    resizeObserver.observe(containerRef.value)
  }

  // 通知主窗口：悬浮窗已就绪
  await emit('screen-banner:ready')
})

onBeforeUnmount(() => {
  unlisten && unlisten()
  if (resizeObserver) resizeObserver.disconnect()
})
</script>

<style>
/* 确保背景透明，无边距 */
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
  /* 这里的样式会被 containerStyle 覆盖，主要作为兜底 */
}
</style>