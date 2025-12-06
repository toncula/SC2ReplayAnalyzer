<template>
  <div class="page">
    <!-- Toolbar -->
    <header class="toolbar">
      <h1 class="title">屏幕提醒</h1>
      <button class="btn" @click="reset">重置</button>
      <button class="btn" :class="{ 'active': !state.locked }" @click="toggleLock">
        {{ state.locked ? '🔒 已锁定' : '🔓 调整位置' }}
      </button>
      <button class="btn primary" @click="toggleShow">
        {{ state.show ? '隐藏悬浮' : '显示悬浮' }} (Ctrl+Shift+S)
      </button>
    </header>

    <div class="content">
      <section class="left">
        <!-- 播放器控制区 -->
        <div class="player-control-panel">
          <div class="player-header">
            <span class="label">流程播放器</span>
            <div class="time-display">
              {{ formatTime(player.currentTime) }} / {{ formatTime(player.maxTime) }}
            </div>
          </div>
          
          <div class="player-controls">
            <button class="btn-icon" @click="togglePlay" :title="player.isPlaying ? '暂停' : '播放'">
              {{ player.isPlaying ? '⏸' : '▶️' }}
            </button>
            <button class="btn-icon" @click="stopPlay" title="停止">⏹</button>
            <input 
              type="range" 
              class="range progress-slider" 
              min="0" 
              :max="player.maxTime" 
              step="1"
              v-model.number="player.currentTime"
              @input="onSeek"
            />
          </div>
          
          <div class="player-settings grid-2">
            <div>
              <label class="sub-label">显示范围 (秒)</label>
              <input type="number" v-model.number="player.windowSize" class="input-sm" />
            </div>
            <div style="display:flex; align-items:center">
              <label class="checkbox-label">
                <input type="checkbox" v-model="player.enabled"> 启用播放模式
              </label>
            </div>
          </div>
        </div>

        <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 5px; margin-top: 15px;">
          <label class="label" style="margin-bottom: 0;">提醒文字 (支持 mm:ss 时间戳)</label>
          <button class="btn-xs" @click="openIconModal">📂 管理/插入图标</button>
        </div>
        
        <textarea
          ref="textareaRef"
          v-model="state.text"
          class="textarea"
          rows="10"
          placeholder="支持时间戳格式：&#10;0:00 [SCV] 开局&#10;0:12 [SCV] 第一个房子&#10;0:45 [Reaper] 死神侦查&#10;..."
          @blur="updateCursorPos"
          @click="updateCursorPos"
          @keyup="updateCursorPos"
        />

        <!-- 样式设置区 -->
        <div class="grid-2">
          <div>
            <label class="label">字体</label>
            <select v-model="state.font.family" class="input">
              <option value="Inter, system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial, 'Apple Color Emoji', 'Segoe UI Emoji'">系统默认</option>
              <option value="'JetBrains Mono', ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, 'Liberation Mono', 'Courier New', monospace">等宽（JetBrains Mono）</option>
              <option value="'Noto Sans SC', 'Microsoft YaHei', 'PingFang SC', system-ui, sans-serif">思源黑体 / 微软雅黑</option>
            </select>
          </div>
          <div>
            <label class="label">粗细</label>
            <select v-model.number="state.font.weight" class="input">
              <option :value="400">正常</option>
              <option :value="600">加粗</option>
            </select>
          </div>
        </div>

        <div class="grid-3">
          <div>
            <label class="label">字号：{{ state.font.size }} px</label>
            <input class="range" type="range" min="12" max="72" v-model.number="state.font.size" />
          </div>
          <div>
            <label class="label">字距：{{ state.style.letterSpacing.toFixed(1) }} px</label>
            <input class="range" type="range" min="-1" max="6" step="0.1" v-model.number="state.style.letterSpacing" />
          </div>
          <div>
            <label class="label">行高：{{ state.style.lineHeight.toFixed(2) }}</label>
            <input class="range" type="range" min="1" max="2" step="0.05" v-model.number="state.style.lineHeight" />
          </div>
        </div>

        <div class="grid-3">
          <div>
            <label class="label">文本颜色</label>
            <input class="input" type="color" v-model="state.colors.text" />
          </div>
          <div>
            <label class="label">背景颜色</label>
            <input class="input" type="color" v-model="state.colors.bg" />
          </div>
          <div>
            <label class="label">背景不透明度：{{ state.style.bgOpacity }}</label>
            <input class="range" type="range" min="0" max="1" step="0.05" v-model.number="state.style.bgOpacity" />
          </div>
        </div>

        <div class="grid-3">
          <div>
            <label class="label">顶部内边距：{{ state.style.paddingY }} px</label>
            <input class="range" type="range" min="4" max="32" step="1" v-model.number="state.style.paddingY" />
          </div>
          <div>
            <label class="label">左右内边距：{{ state.style.paddingX }} px</label>
            <input class="range" type="range" min="8" max="64" step="2" v-model.number="state.style.paddingX" />
          </div>
          <div>
            <label class="label">阴影</label>
            <select v-model="state.style.shadow" class="input">
              <option value="none">无</option>
              <option value="sm">浅</option>
              <option value="md">中</option>
              <option value="lg">重</option>
            </select>
          </div>
        </div>

        <!-- 
        <div class="grid-3" style="margin-top: 12px;">
          <div>
            <label class="label">窗口宽度（px）</label>
            <input class="input" type="number" min="400" max="9999" v-model.number="state.window.width" />
          </div>
          <div>
            <label class="label">窗口高度（px）</label>
            <input class="input" type="number" min="40" max="600" v-model.number="state.window.height" />
          </div>
          <div>
            <label class="label">宽度模式</label>
            <select v-model="state.layout.width" class="input">
              <option value="full">全宽</option>
              <option value="center">居中（最大 960px）</option>
            </select>
          </div>
        </div> -->

        <!-- <div class="grid-3" style="margin-top: 12px;">
          <div>
            <label class="label">层级 z-index</label>
            <input class="input" type="number" v-model.number="state.layout.zIndex" min="10" max="999999" />
          </div>
          <div>
            <label class="label">交互</label>
            <select v-model="state.layout.pointer" class="input">
              <option value="auto">可点击</option>
              <option value="none">穿透（不拦截鼠标）</option>
            </select>
          </div>
          <div>
            <label class="label">启用 N 秒后闪烁</label>
            <select v-model="state.behavior.blinkEnabled" class="input">
              <option :value="true">启用</option>
              <option :value="false">禁用</option>
            </select>
          </div>
        </div>

        <div class="grid-3" style="margin-top: 12px;">
          <div>
            <label class="label">N 秒后开始闪烁：{{ state.behavior.blinkAfter }} s</label>
            <input class="range" type="range" min="10" max="600" step="5" v-model.number="state.behavior.blinkAfter" />
          </div>
          <div>
            <label class="label">（可选）闪烁持续秒数：{{ state.behavior.blinkDuration }} s</label>
            <input class="range" type="range" min="5" max="120" step="5" v-model.number="state.behavior.blinkDuration" />
          </div>
          <div style="display:flex;align-items:flex-end;">
            <span class="label">
              全局快捷键：
              <code>`</code> 重置闪烁倒计时
            </span>
          </div>
        </div> -->

        <!-- <div class="tips">
          快捷键：
          <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>S</kbd> 显示/隐藏；
          <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>↑/↓</kbd> 调整字号；
          <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>C</kbd> 复制当前文字；
          <code>`</code>（全局）重置即将闪烁的时间。
        </div> -->
        
          <div style="margin-top: 20px; font-size: 12px; color: #666; line-height: 1.5;">
            提示：<br/>
            1. 只有点击上方 <b>"解锁位置"</b> 后，悬浮窗才可以被鼠标选中并拖动。<br/>
            2. 勾选 <b>"启用播放模式"</b> 后，悬浮窗底部会出现进度条和控制按钮。<br/>
            3. <b>注意：</b>当悬浮窗处于<b>锁定</b>状态时，无法点击进度条（鼠标穿透）。如需操作，请先解锁或使用主界面控制。
         </div>
      </section>

      <section class="right">
        <div class="preview-title">预览 (所见即所得)</div>
        <div class="preview">
          <div class="banner" :style="bannerStyle">
            <div v-html="parsedHtmlText"></div>
            <!-- 预览区显示模拟进度条 -->
            <div v-if="player.enabled" class="preview-controls" style="margin-top:8px; border-top:1px solid rgba(255,255,255,0.1); padding-top:4px;">
               <div style="display:flex; gap:4px; align-items:center;">
                 <div style="font-size:12px;">⏸</div>
                 <div style="flex:1; height:4px; background:rgba(255,255,255,0.2); border-radius:2px;">
                   <div :style="{ width: (player.maxTime > 0 ? (player.currentTime / player.maxTime) * 100 : 0) + '%', height: '100%', background: '#3b82f6' }"></div>
                 </div>
                 <div style="font-size:10px; opacity:0.7;">{{ formatTime(player.currentTime) }}</div>
               </div>
            </div>
          </div>
        </div>
      </section>
    </div>

    <!-- 图标管理模态框 -->
    <div v-if="showIconModal" class="modal-overlay" @click.self="showIconModal = false">
      <div class="modal-content">
        <div class="modal-header">
          <h3>自定义图标库</h3>
          <div style="display:flex; gap: 10px;">
            <input type="file" ref="fileInputRef" accept="image/*" multiple style="display: none" @change="handleFileUpload" />
            <button class="btn primary" @click="triggerFileUpload">📤 上传图标</button>
            <button class="close-btn" @click="showIconModal = false">×</button>
          </div>
        </div>
        <div class="modal-body">
          <div v-if="userIcons.length === 0" class="empty-state">暂无图标，请上传。</div>
          <div v-else class="icon-grid">
            <div v-for="(icon, index) in userIcons" :key="index" class="icon-card">
              <div class="img-wrapper" @click="insertIcon(icon.name)" title="点击插入">
                <img :src="icon.src" loading="lazy" />
              </div>
              <div class="edit-wrapper">
                <span class="bracket">[</span>
                <input v-model="icon.name" class="name-input" @change="saveIconsToStorage" />
                <span class="bracket">]</span>
                <button class="delete-btn" @click="deleteIcon(index)" title="删除">×</button>
              </div>
            </div>
          </div>
        </div>
        <div class="modal-footer">已存储 {{ userIcons.length }} 个图标 (本地缓存)</div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { reactive, computed, onMounted, onUnmounted, watch, ref } from 'vue'
import { WebviewWindow } from '@tauri-apps/api/webviewWindow'
import { listen, emit } from '@tauri-apps/api/event'
import { register as registerShortcut, unregisterAll } from '@tauri-apps/plugin-global-shortcut'

// 1. 工具函数
const shadowMap: Record<string, string> = { none: 'none', sm: '0 1px 2px rgba(0,0,0,.25)', md: '0 6px 16px rgba(0,0,0,.35)', lg: '0 14px 28px rgba(0,0,0,.45)' }
function hexToRgba(hex: string, a: number) { const m = hex.replace('#',''); const full = m.length === 3 ? m.split('').map(ch => ch + ch).join('') : m; const bigint = parseInt(full, 16); const r = (bigint >> 16) & 255; const g = (bigint >> 8) & 255; const b = bigint & 255; return `rgba(${r}, ${g}, ${b}, ${a})` }
function formatTime(s: number) { const m = Math.floor(s / 60); const sec = Math.floor(s % 60); return `${m}:${sec.toString().padStart(2, '0')}` }
function escapeHtml(t: string) { return t.replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;").replace(/"/g, "&quot;").replace(/'/g, "&#039;") }
function toBase64(file: File) { return new Promise((res, rej) => { const r = new FileReader(); r.readAsDataURL(file); r.onload = () => res(r.result); r.onerror = rej }) }
const STORE_KEY = 'screen-reminder-v1'; const ICONS_STORE_KEY = 'user-custom-icons';

// 2. 状态定义
const player = reactive({ enabled: false, isPlaying: false, currentTime: 0, maxTime: 600, windowSize: 5, timer: null as any })
const state = reactive({
  text: '', show: false, locked: true,
  font: { size: 28, family: "Inter, system-ui, sans-serif", weight: 700 },
  colors: { text: '#ffffff', bg: '#111827' },
  style: { letterSpacing: 0.2, lineHeight: 1.2, paddingX: 24, paddingY: 10, bgOpacity: 0.85, shadow: 'md' as any },
  layout: { width: 'full', zIndex: 99999, pointer: 'none' },
  window: { width: 1920, height: 60 },
  behavior: { blinkEnabled: true, blinkAfter: 180, blinkDuration: 30 },
})

interface UserIcon { name: string; src: string; }
const showIconModal = ref(false)
const userIcons = ref<UserIcon[]>([])
const fileInputRef = ref<HTMLInputElement | null>(null)
const textareaRef = ref<HTMLTextAreaElement | null>(null)
const cursorPosition = ref(0)

const BANNER_LABEL = 'screen-banner'
let bannerWin: WebviewWindow | null = null

// 3. 核心计算
// 只负责解析文本，不生成进度条HTML (进度条由悬浮窗组件自己生成)
const parsedHtmlText = computed(() => {
  if (!state.text) return '示例提示文字'
  let linesToDisplay: string[] = []
  const safeText = escapeHtml(state.text)
  const rawLines = safeText.split('\n')

  if (player.enabled) {
    rawLines.forEach(line => {
      const match = line.match(/^\s*(\d+)[:：](\d+)/)
      if (match) {
        const t = parseInt(match[1]) * 60 + parseInt(match[2])
        if (t >= player.currentTime - player.windowSize && t <= player.currentTime + player.windowSize) {
          if (t === player.currentTime) linesToDisplay.push(`<span style="color:#3b82f6;font-weight:bold;">${line}</span>`)
          else linesToDisplay.push(line)
        }
      }
    })
    if (linesToDisplay.length === 0) linesToDisplay.push(`<span style="opacity:0.5;font-size:0.8em">(${formatTime(player.currentTime)}) 等待指令...</span>`)
  } else {
    linesToDisplay = rawLines
  }

  return linesToDisplay.join('<br/>').replace(/\[([a-zA-Z0-9_\-\u4e00-\u9fa5\s]+)\]/g, (m, k) => {
    const icon = userIcons.value.find(i => i.name === k.trim())
    return icon ? `<img src="${icon.src}" style="height:1.2em;vertical-align:text-bottom;margin:0 1px;"/>` : m
  })
})

const bannerStyle = computed(() => ({
  color: state.colors.text,
  background: hexToRgba(state.colors.bg, state.style.bgOpacity),
  fontSize: state.font.size + 'px',
  fontFamily: state.font.family,
  fontWeight: String(state.font.weight),
  letterSpacing: state.style.letterSpacing + 'px',
  lineHeight: String(state.style.lineHeight),
  padding: `${state.style.paddingY}px ${state.style.paddingX}px`,
  boxShadow: shadowMap[state.style.shadow],
  whiteSpace: 'pre-wrap' as const,
  maxWidth: '1200px',
  width: 'fit-content',
  minWidth: player.enabled ? '300px' : '0', // 播放模式下保证最小宽度
  overflow: 'hidden',
  textOverflow: 'ellipsis',
  cursor: (state.locked ? 'default' : 'move') as 'default' | 'move',
  pointerEvents: (state.locked ? 'none' : 'auto') as 'none' | 'auto',
}))

// 4. 事件发送
function emitUpdate() {
  emit('screen-banner:update', {
    text: parsedHtmlText.value,
    style: bannerStyle.value,
    behavior: {
      player: {
        enabled: player.enabled,
        isPlaying: player.isPlaying,
        currentTime: player.currentTime,
        maxTime: player.maxTime
      }
    },
    locked: state.locked
  })
}

// 5. 播放逻辑
function togglePlay() {
  if (player.isPlaying) stopTimer()
  else {
    player.enabled = true; player.isPlaying = true
    player.timer = setInterval(() => {
      if (player.currentTime >= player.maxTime) stopPlay()
      else player.currentTime++
    }, 1000)
  }
}
function stopPlay() { stopTimer(); player.currentTime = 0 }
function stopTimer() { player.isPlaying = false; if (player.timer) { clearInterval(player.timer); player.timer = null } }
function onSeek() { if (player.isPlaying) { stopTimer(); togglePlay() } }

// 6. 窗口管理
async function openBannerWindow() {
  if (bannerWin) { try { await bannerWin.show(); return } catch { bannerWin = null } }
  bannerWin = new WebviewWindow(BANNER_LABEL, {
    url: '/floating-banner',
    width: 400, height: 100, x: 0, y: 0,
    decorations: false, transparent: true, alwaysOnTop: true, resizable: false, focus: false, shadow: false
  })
  bannerWin.once('tauri://created', async () => { try { await updateLockState() } catch(e){} })
  bannerWin.once('tauri://destroyed', () => { bannerWin = null; state.show = false })
}
async function closeBannerWindow() { if (!bannerWin) return; try { await bannerWin.close() } catch(e){} finally { bannerWin = null } }
async function toggleShow() { state.show = !state.show; if (state.show) { saveToLocal(); await openBannerWindow(); setTimeout(emitUpdate, 1000) } else { await closeBannerWindow() } }
async function toggleLock() { state.locked = !state.locked; await updateLockState(); emitUpdate() }
async function updateLockState() { if (!bannerWin) return; await bannerWin.setIgnoreCursorEvents(state.locked); if (!state.locked) await bannerWin.setFocus() }

// 7. 其他辅助
function openIconModal() { showIconModal.value = true }
function triggerFileUpload() { fileInputRef.value?.click() }
async function handleFileUpload(e: Event) { /* ...同前... */ 
  const target = e.target as HTMLInputElement; const files = target.files; if(!files) return
  for(let i=0; i<files.length; i++) {
    const f = files[i]; let n = f.name.split('.')[0].toLowerCase().replace(/\s+/g,'_')
    let c=1; let tn=n; while(userIcons.value.some(x=>x.name===tn)) { tn=`${n}_${c}`; c++ }
    userIcons.value.push({ name: tn, src: (await toBase64(f)) as string })
  }
  saveIconsToStorage(); target.value=''
}
function deleteIcon(i: number) { if(confirm('Del?')) { userIcons.value.splice(i,1); saveIconsToStorage() } }
function saveIconsToStorage() { localStorage.setItem(ICONS_STORE_KEY, JSON.stringify(userIcons.value)) }
function loadIconsFromStorage() { const r = localStorage.getItem(ICONS_STORE_KEY); if(r) userIcons.value = JSON.parse(r) }
function updateCursorPos() { if (textareaRef.value) cursorPosition.value = textareaRef.value.selectionStart }
function insertIcon(tag: string) { /* ...同前... */ 
  const ins = `[${tag}]`; const val = state.text; const p = cursorPosition.value
  state.text = val.slice(0,p) + ins + val.slice(p); cursorPosition.value += ins.length
  showIconModal.value=false
}
function reset() { state.text=''; state.font.size=28; state.colors.text='#ffffff'; state.colors.bg='#111827'; state.locked=true; updateLockState() }
function saveToLocal() { localStorage.setItem(STORE_KEY, JSON.stringify(state)) }
function loadFromLocal() { 
  const r = localStorage.getItem(STORE_KEY); 
  if(r) { 
    const p = JSON.parse(r); 
    if(p.text!==undefined) state.text=p.text; 
    if(p.font) Object.assign(state.font, p.font); 
    if(p.colors) Object.assign(state.colors, p.colors); 
    if(p.style) Object.assign(state.style, p.style); 
    state.show = false 
  } 
}
function onKey(e: KeyboardEvent) { 
  if(!e.ctrlKey && !e.shiftKey && e.code==='Backquote') emit('screen-banner:resetBlink')
  if(e.ctrlKey && e.shiftKey && e.code==='KeyS') { e.preventDefault(); toggleShow() }
}

onMounted(async () => {
  loadFromLocal(); loadIconsFromStorage(); window.addEventListener('keydown', onKey)
  await listen('screen-banner:ready', () => emitUpdate())
  await listen('screen-banner:control', (e: any) => {
    if (e.payload.type === 'toggle') togglePlay()
    if (e.payload.type === 'seek') { player.currentTime = e.payload.value; onSeek() }
  })
})
onUnmounted(() => { stopTimer(); window.removeEventListener('keydown', onKey) })

watch(state, () => { saveToLocal(); emitUpdate() }, { deep: true })
watch(() => state.text, (val) => { 
  const lines = val.split('\n'); let max=0
  lines.forEach(l => { const m = l.match(/^\s*(\d+)[:：](\d+)/); if(m) { const t=parseInt(m[1])*60+parseInt(m[2]); if(t>max) max=t } })
  if(max>0) player.maxTime = max + 10
})
watch(() => player.currentTime, emitUpdate)
watch(() => player.enabled, emitUpdate)
watch(() => player.isPlaying, emitUpdate)
</script>

<style scoped>
/* 保持所有样式 */
.page { display: flex; flex-direction: column; height: 100%; color: #e5e7eb; background: #0f1113; }
.toolbar h1 { font-size: 18px; font-weight: 700; }
.content { display:grid; grid-template-columns: 420px 1fr; gap:16px; padding:16px 20px; height: calc(100vh - 50px); margin-top: 50px; box-sizing:border-box; overflow:hidden; }
.left, .right { background: #161a1f; border: 1px solid #23272e; border-radius: 14px; padding: 14px; overflow: auto; }
.label { font-size: 12px; color: #a6adbb; margin-bottom: 6px; display: block; }
.input, .textarea, .range { width: 100%; box-sizing: border-box; }
.input, .textarea { background: #0f1317; color: #e5e7eb; border: 1px solid #2a2f36; border-radius: 10px; padding: 10px 12px; }
.textarea { resize: vertical; }
.range { accent-color: #3b82f6; }
.grid-2 { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin-top: 12px; }
.grid-3 { display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 10px; margin-top: 12px; }
.btn { border: 1px solid #2a2f36; background: #0f1317; color: #e5e7eb; padding: 4px 12px; border-radius: 10px; cursor: pointer; }
.btn:hover { filter: brightness(1.1); }
.btn.primary { background: #3b82f6; border-color: transparent; color: white; }
.preview-title { font-size: 12px; color: #a6adbb; margin-bottom: 8px; }
.preview { border: 1px dashed #2a2f36; border-radius: 12px; padding: 10px; background: #0f1113; }
.banner { width: 100%; text-align: left; border-radius: 10px; }
.player-control-panel { background: #1f242d; border: 1px solid #374151; border-radius: 10px; padding: 12px; margin-bottom: 16px; }
.player-header { display: flex; justify-content: space-between; margin-bottom: 8px; }
.time-display { font-family: monospace; font-size: 14px; color: #3b82f6; }
.player-controls { display: flex; gap: 8px; align-items: center; margin-bottom: 8px; }
.btn-icon { background: none; border: 1px solid #374151; color: #e5e7eb; border-radius: 4px; width: 32px; height: 32px; cursor: pointer; display: flex; align-items: center; justify-content: center; }
.btn-icon:hover { background: #374151; }
.progress-slider { flex: 1; }
.player-settings { margin-top: 8px; }
.sub-label { font-size: 11px; color: #6b7280; display: block; margin-bottom: 2px; }
.input-sm { width: 60px; padding: 4px; font-size: 12px; background: #111827; border: 1px solid #374151; border-radius: 4px; color: white; }
.checkbox-label { font-size: 12px; cursor: pointer; display: flex; align-items: center; gap: 6px; }
.btn-xs { border: 1px solid #3b82f6; background: rgba(59, 130, 246, 0.1); color: #3b82f6; padding: 2px 8px; border-radius: 6px; cursor: pointer; font-size: 11px; }
.btn-xs:hover { background: rgba(59, 130, 246, 0.2); }
.modal-overlay { position: fixed; top: 0; left: 0; right: 0; bottom: 0; background: rgba(0,0,0,0.7); display: flex; justify-content: center; align-items: center; z-index: 10000; }
.modal-content { background: #1f242d; width: 700px; max-height: 80vh; border-radius: 12px; border: 1px solid #374151; display: flex; flex-direction: column; }
.modal-header { padding: 16px; border-bottom: 1px solid #374151; display: flex; justify-content: space-between; align-items: center; }
.modal-body { padding: 20px; overflow-y: auto; flex: 1; }
.modal-footer { padding: 10px 20px; border-top: 1px solid #374151; font-size: 11px; color: #6b7280; text-align: right; }
.close-btn { background: none; border: none; color: #9ca3af; cursor: pointer; font-size: 20px; }
.empty-state { text-align: center; color: #6b7280; padding: 40px; }
.icon-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(120px, 1fr)); gap: 16px; }
.icon-card { background: #262b36; border: 1px solid #374151; border-radius: 8px; padding: 8px; display: flex; flex-direction: column; align-items: center; }
.icon-card:hover { border-color: #4b5563; }
.img-wrapper { width: 100%; height: 60px; display: flex; align-items: center; justify-content: center; cursor: pointer; background: #161a1f; border-radius: 4px; margin-bottom: 8px; }
.img-wrapper img { max-width: 100%; max-height: 100%; object-fit: contain; }
.edit-wrapper { display: flex; align-items: center; width: 100%; font-size: 12px; color: #9ca3af; }
.bracket { opacity: 0.5; font-family: monospace; }
.name-input { background: transparent; border: none; border-bottom: 1px dashed #4b5563; color: #e5e7eb; font-family: monospace; width: 100%; text-align: center; margin: 0 4px; font-size: 11px; }
.name-input:focus { outline: none; border-bottom: 1px solid #3b82f6; color: #3b82f6; }
.delete-btn { background: none; border: none; color: #ef4444; cursor: pointer; font-size: 16px; opacity: 0.5; margin-left: 2px; }
.delete-btn:hover { opacity: 1; }
</style>