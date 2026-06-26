<template>
  <div class="chat-page">

    <!-- 顶栏 -->
    <header class="topbar">
      <button class="back-btn" @click="goBack">
        <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
          <path d="M19 12H5M12 5l-7 7 7 7"/>
        </svg>
      </button>

      <div class="topbar-center">
        <div class="ai-badge">
          <div class="badge-avatar">
            <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <path d="M20.84 4.61a5.5 5.5 0 0 0-7.78 0L12 5.67l-1.06-1.06a5.5 5.5 0 0 0-7.78 7.78l1.06 1.06L12 21.23l7.78-7.78 1.06-1.06a5.5 5.5 0 0 0 0-7.78z"/>
            </svg>
          </div>
          <span class="badge-name">恋爱大师</span>
          <div class="status-dot" :class="{ active: connectionStatus === 'connecting' }"></div>
        </div>
      </div>

      <button class="new-chat-btn" @click="newChat" title="新对话">
        <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <path d="M12 5v14M5 12h14"/>
        </svg>
      </button>
    </header>

    <!-- 消息区 -->
    <main class="messages-area" ref="messagesEl">
      <div class="messages-inner">

        <!-- 空状态 -->
        <transition name="fade">
          <div v-if="messages.length === 0" class="empty-state">
            <div class="empty-avatar">
              <svg width="28" height="28" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
                <path d="M20.84 4.61a5.5 5.5 0 0 0-7.78 0L12 5.67l-1.06-1.06a5.5 5.5 0 0 0-7.78 7.78l1.06 1.06L12 21.23l7.78-7.78 1.06-1.06a5.5 5.5 0 0 0 0-7.78z"/>
              </svg>
            </div>
            <h2 class="empty-title">有什么心事，说出来吧</h2>
            <p class="empty-sub">我会帮你分析感情问题，给出真诚的建议</p>
            <div class="prompt-chips">
              <button
                v-for="p in promptSuggestions"
                :key="p"
                class="chip"
                @click="usePrompt(p)"
              >{{ p }}</button>
            </div>
          </div>
        </transition>

        <!-- 消息列表 -->
        <TransitionGroup name="msg" tag="div" class="message-list">
          <div
            v-for="(msg, i) in messages"
            :key="msg.id"
            class="message-row"
            :class="msg.isUser ? 'row-user' : 'row-ai'"
            @mouseenter="hoveredId = msg.id"
            @mouseleave="hoveredId = null"
          >

            <!-- AI 行 -->
            <template v-if="!msg.isUser">
              <div class="ai-avatar-wrap">
                <div class="ai-avatar">
                  <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                    <path d="M20.84 4.61a5.5 5.5 0 0 0-7.78 0L12 5.67l-1.06-1.06a5.5 5.5 0 0 0-7.78 7.78l1.06 1.06L12 21.23l7.78-7.78 1.06-1.06a5.5 5.5 0 0 0 0-7.78z"/>
                  </svg>
                </div>
              </div>
              <div class="ai-content">
                <div class="ai-name">恋爱大师</div>
                <div class="ai-text">
                  <span v-if="msg.content">{{ msg.content }}</span>
                  <span v-else-if="connectionStatus === 'connecting' && i === messages.length - 1" class="typing-dots">
                    <span></span><span></span><span></span>
                  </span>
                  <span
                    v-if="msg.content && connectionStatus === 'connecting' && i === messages.length - 1"
                    class="cursor-blink"
                  >▋</span>
                </div>
                <div class="ai-actions" :class="{ visible: hoveredId === msg.id && msg.content }">
                  <button class="action-btn" @click="copyMsg(msg.content)" :title="copied === msg.id ? '已复制' : '复制'">
                    <svg v-if="copied !== msg.id" width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                      <rect x="9" y="9" width="13" height="13" rx="2"/><path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1"/>
                    </svg>
                    <svg v-else width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="#22c55e" stroke-width="2.5">
                      <path d="M20 6L9 17l-5-5"/>
                    </svg>
                  </button>
                  <span class="msg-time">{{ formatTime(msg.time) }}</span>
                </div>
              </div>
            </template>

            <!-- 用户行 -->
            <template v-else>
              <div class="user-content">
                <div class="user-bubble">{{ msg.content }}</div>
                <transition name="fade">
                  <div class="user-time" v-if="hoveredId === msg.id">{{ formatTime(msg.time) }}</div>
                </transition>
              </div>
              <div class="user-avatar">
                <svg width="13" height="13" viewBox="0 0 24 24" fill="currentColor">
                  <path d="M12 12c2.7 0 4.8-2.1 4.8-4.8S14.7 2.4 12 2.4 7.2 4.5 7.2 7.2 9.3 12 12 12zm0 2.4c-3.2 0-9.6 1.6-9.6 4.8v2.4h19.2v-2.4c0-3.2-6.4-4.8-9.6-4.8z"/>
                </svg>
              </div>
            </template>

          </div>
        </TransitionGroup>

        <!-- 底部锚点 -->
        <div ref="bottomEl" style="height:1px"></div>
      </div>
    </main>

    <!-- 输入区 -->
    <footer class="input-area">
      <div class="input-card" :class="{ focused: inputFocused, sending: connectionStatus === 'connecting' }">
        <textarea
          ref="textareaEl"
          v-model="inputText"
          class="input-textarea"
          placeholder="和恋爱大师聊聊…"
          :disabled="connectionStatus === 'connecting'"
          @focus="inputFocused = true"
          @blur="inputFocused = false"
          @keydown.enter.exact.prevent="send"
          @keydown.shift.enter.exact="newline"
          @input="autoResize"
          rows="1"
        ></textarea>

        <div class="input-footer-row">
          <span class="input-hint-inner">Shift+Enter 换行</span>
          <button
            class="send-btn"
            :class="{ ready: canSend }"
            :disabled="!canSend"
            @click="send"
          >
            <transition name="icon-swap" mode="out-in">
              <svg v-if="connectionStatus !== 'connecting'" key="send" width="16" height="16" viewBox="0 0 24 24" fill="currentColor">
                <path d="M2.01 21L23 12 2.01 3 2 10l15 2-15 2z"/>
              </svg>
              <svg v-else key="loading" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" class="spin">
                <path d="M12 2v4M12 18v4M4.93 4.93l2.83 2.83M16.24 16.24l2.83 2.83M2 12h4M18 12h4M4.93 19.07l2.83-2.83M16.24 7.76l2.83-2.83"/>
              </svg>
            </transition>
          </button>
        </div>
      </div>
      <p class="powered-by">由 通义千问 · Qwen 提供支持</p>
    </footer>

  </div>
</template>

<script setup>
import { ref, computed, nextTick, onMounted, onBeforeUnmount } from 'vue'
import { useRouter } from 'vue-router'
import { useHead } from '@vueuse/head'
import { chatWithLoveApp } from '../api'

useHead({ title: 'AI 恋爱大师' })

const router = useRouter()
const messages   = ref([])
const chatId     = ref('')
const connectionStatus = ref('disconnected')
const inputText  = ref('')
const inputFocused = ref(false)
const messagesEl = ref(null)
const textareaEl = ref(null)
const bottomEl   = ref(null)
const hoveredId  = ref(null)
const copied     = ref(null)
let eventSource  = null
let msgId        = 0

const promptSuggestions = [
  '我喜欢一个人，但不知道他是否有意思',
  '和另一半吵架了，该怎么和解',
  '异地恋维持不下去了，怎么办',
  '怎么才能让自己更有魅力',
]

const canSend = computed(() =>
  inputText.value.trim().length > 0 && connectionStatus.value !== 'connecting'
)

const generateChatId = () => 'love_' + Math.random().toString(36).slice(2, 10)

const formatTime = (ts) =>
  new Date(ts).toLocaleTimeString('zh-CN', { hour: '2-digit', minute: '2-digit' })

const scrollToBottom = async () => {
  await nextTick()
  bottomEl.value?.scrollIntoView({ behavior: 'smooth' })
}

const autoResize = () => {
  const el = textareaEl.value
  if (!el) return
  el.style.height = 'auto'
  el.style.height = Math.min(el.scrollHeight, 180) + 'px'
}

const newline = () => {
  inputText.value += '\n'
  nextTick(autoResize)
}

const addMsg = (content, isUser) => {
  messages.value.push({ id: ++msgId, content, isUser, time: Date.now() })
  scrollToBottom()
}

const usePrompt = (text) => {
  inputText.value = text
  nextTick(() => { textareaEl.value?.focus(); autoResize() })
}

const copyMsg = async (text) => {
  try {
    await navigator.clipboard.writeText(text)
    copied.value = msgId
    const id = messages.value.find(m => m.content === text)?.id
    copied.value = id
    setTimeout(() => { copied.value = null }, 2000)
  } catch {}
}

const send = () => {
  const text = inputText.value.trim()
  if (!canSend.value) return

  addMsg(text, true)
  inputText.value = ''
  nextTick(() => { autoResize(); textareaEl.value?.focus() })

  if (eventSource) eventSource.close()

  const aiIndex = messages.value.length
  addMsg('', false)
  connectionStatus.value = 'connecting'

  eventSource = chatWithLoveApp(text, chatId.value)

  eventSource.onmessage = (e) => {
    const data = e.data
    if (data && data !== '[DONE]') {
      if (aiIndex < messages.value.length) {
        messages.value[aiIndex].content += data
        scrollToBottom()
      }
    }
    if (data === '[DONE]') {
      connectionStatus.value = 'disconnected'
      eventSource.close()
    }
  }

  eventSource.onerror = () => {
    connectionStatus.value = 'error'
    eventSource.close()
    if (messages.value[aiIndex]?.content === '') {
      messages.value[aiIndex].content = '连接出现问题，请稍后重试。'
    }
  }
}

const newChat = () => {
  if (eventSource) eventSource.close()
  messages.value = []
  chatId.value = generateChatId()
  connectionStatus.value = 'disconnected'
}

const goBack = () => router.push('/')

onMounted(() => {
  chatId.value = generateChatId()
})

onBeforeUnmount(() => {
  if (eventSource) eventSource.close()
})
</script>

<style scoped>
/* ── 基础布局 ── */
*, *::before, *::after { box-sizing: border-box; }

.chat-page {
  display: flex;
  flex-direction: column;
  height: 100dvh;
  background: #fff;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'PingFang SC', sans-serif;
  color: #111;
  -webkit-font-smoothing: antialiased;
}

/* ── 顶栏 ── */
.topbar {
  display: grid;
  grid-template-columns: 48px 1fr 48px;
  align-items: center;
  height: 52px;
  padding: 0 12px;
  border-bottom: 1px solid #f0f0f0;
  flex-shrink: 0;
  position: sticky;
  top: 0;
  background: rgba(255,255,255,0.92);
  backdrop-filter: blur(16px);
  z-index: 20;
}

.back-btn, .new-chat-btn {
  width: 36px;
  height: 36px;
  border-radius: 10px;
  border: none;
  background: none;
  cursor: pointer;
  color: #666;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.15s, color 0.15s;
}
.back-btn:hover, .new-chat-btn:hover {
  background: #f4f4f4;
  color: #111;
}
.new-chat-btn { justify-self: end; }

.topbar-center { display: flex; justify-content: center; }

.ai-badge {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 5px 12px 5px 6px;
  border-radius: 20px;
  background: #f8f7f5;
  border: 1px solid #ebebeb;
}

.badge-avatar {
  width: 26px;
  height: 26px;
  border-radius: 50%;
  background: linear-gradient(135deg, #6366f1, #8b5cf6);
  display: flex;
  align-items: center;
  justify-content: center;
  color: #fff;
  flex-shrink: 0;
}

.badge-name {
  font-size: 13px;
  font-weight: 600;
  color: #222;
}

.status-dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: #d1d5db;
  transition: background 0.3s;
  flex-shrink: 0;
}
.status-dot.active {
  background: #22c55e;
  animation: pulse 1.5s infinite;
}
@keyframes pulse {
  0%,100% { box-shadow: 0 0 0 0 rgba(34,197,94,0.35); }
  50%      { box-shadow: 0 0 0 5px rgba(34,197,94,0); }
}

/* ── 消息区 ── */
.messages-area {
  flex: 1;
  overflow-y: auto;
  overscroll-behavior: contain;
}
.messages-area::-webkit-scrollbar { width: 4px; }
.messages-area::-webkit-scrollbar-thumb { background: #e5e7eb; border-radius: 4px; }

.messages-inner {
  max-width: 720px;
  margin: 0 auto;
  padding: 28px 20px 8px;
}

/* ── 空状态 ── */
.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 64px 0 40px;
  text-align: center;
}

.empty-avatar {
  width: 56px;
  height: 56px;
  border-radius: 50%;
  background: linear-gradient(135deg, #6366f1, #8b5cf6);
  display: flex;
  align-items: center;
  justify-content: center;
  color: #fff;
  margin-bottom: 20px;
  box-shadow: 0 8px 24px rgba(99,102,241,0.25);
}

.empty-title {
  font-size: 20px;
  font-weight: 700;
  color: #111;
  margin: 0 0 8px;
}

.empty-sub {
  font-size: 14px;
  color: #9ca3af;
  margin: 0 0 28px;
}

.prompt-chips {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  justify-content: center;
  max-width: 500px;
}

.chip {
  padding: 8px 16px;
  border-radius: 20px;
  border: 1px solid #e5e7eb;
  background: #fafafa;
  color: #374151;
  font-size: 13px;
  cursor: pointer;
  transition: border-color 0.15s, background 0.15s, transform 0.1s;
  line-height: 1.4;
}
.chip:hover {
  border-color: #6366f1;
  background: #f5f3ff;
  color: #4f46e5;
  transform: translateY(-1px);
}

/* ── 消息列表 ── */
.message-list {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.message-row {
  display: flex;
  gap: 12px;
  padding: 6px 0;
}
.row-ai   { align-items: flex-start; }
.row-user { flex-direction: row-reverse; align-items: flex-end; }

/* AI 消息 */
.ai-avatar-wrap { flex-shrink: 0; padding-top: 2px; }

.ai-avatar {
  width: 30px;
  height: 30px;
  border-radius: 50%;
  background: linear-gradient(135deg, #6366f1, #8b5cf6);
  display: flex;
  align-items: center;
  justify-content: center;
  color: #fff;
  flex-shrink: 0;
}

.ai-content {
  flex: 1;
  min-width: 0;
}

.ai-name {
  font-size: 12px;
  font-weight: 600;
  color: #9ca3af;
  margin-bottom: 4px;
  letter-spacing: 0.01em;
}

.ai-text {
  display: inline-block;
  background: #f4f3f0;
  border-radius: 4px 18px 18px 18px;
  padding: 12px 16px;
  font-size: 15px;
  line-height: 1.75;
  color: #111;
  white-space: pre-wrap;
  word-break: break-word;
  max-width: min(72%, 520px);
}

.ai-actions {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-top: 6px;
  opacity: 0;
  transform: translateY(2px);
  transition: opacity 0.15s, transform 0.15s;
  pointer-events: none;
}
.ai-actions.visible {
  opacity: 1;
  transform: none;
  pointer-events: auto;
}

.action-btn {
  width: 26px;
  height: 26px;
  border-radius: 6px;
  border: 1px solid #e5e7eb;
  background: #fff;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #6b7280;
  transition: background 0.15s, border-color 0.15s;
}
.action-btn:hover { background: #f9fafb; border-color: #d1d5db; }

.msg-time {
  font-size: 11px;
  color: #d1d5db;
}

/* 用户消息 */
.user-content {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  max-width: min(70%, 520px);
  gap: 4px;
}

.user-bubble {
  background: #111;
  color: #f9fafb;
  padding: 10px 16px;
  border-radius: 18px 18px 4px 18px;
  font-size: 15px;
  line-height: 1.65;
  word-break: break-word;
  white-space: pre-wrap;
}

.user-time {
  font-size: 11px;
  color: #d1d5db;
  padding-right: 4px;
}

.user-avatar {
  width: 30px;
  height: 30px;
  border-radius: 50%;
  background: #f3f4f6;
  border: 1px solid #e5e7eb;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #6b7280;
  flex-shrink: 0;
}

/* ── 打字动效 ── */
.typing-dots {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  height: 20px;
}
.typing-dots span {
  width: 5px;
  height: 5px;
  background: #c4b5fd;
  border-radius: 50%;
  animation: bounce 1.2s infinite;
}
.typing-dots span:nth-child(2) { animation-delay: 0.18s; }
.typing-dots span:nth-child(3) { animation-delay: 0.36s; }
@keyframes bounce {
  0%,80%,100% { transform: translateY(0); opacity: 0.5; }
  40%          { transform: translateY(-6px); opacity: 1; }
}

.cursor-blink {
  display: inline-block;
  width: 2px;
  height: 16px;
  background: #6366f1;
  border-radius: 1px;
  margin-left: 2px;
  vertical-align: middle;
  animation: blink 0.8s infinite;
}
@keyframes blink { 0%,100% { opacity: 0; } 50% { opacity: 1; } }

/* ── 输入区 ── */
.input-area {
  padding: 12px 20px 16px;
  background: #fff;
  border-top: 1px solid #f0f0f0;
  flex-shrink: 0;
}

.input-card {
  max-width: 720px;
  margin: 0 auto;
  background: #fafafa;
  border: 1.5px solid #e5e7eb;
  border-radius: 18px;
  padding: 12px 12px 8px 16px;
  transition: border-color 0.2s, box-shadow 0.2s;
}
.input-card.focused {
  border-color: #6366f1;
  box-shadow: 0 0 0 3px rgba(99,102,241,0.1);
  background: #fff;
}
.input-card.sending { opacity: 0.7; }

.input-textarea {
  width: 100%;
  border: none;
  outline: none;
  background: transparent;
  font-size: 15px;
  line-height: 1.6;
  resize: none;
  color: #111;
  max-height: 180px;
  overflow-y: auto;
  font-family: inherit;
  display: block;
}
.input-textarea::placeholder { color: #c4c4c4; }
.input-textarea::-webkit-scrollbar { width: 3px; }
.input-textarea::-webkit-scrollbar-thumb { background: #e5e7eb; border-radius: 3px; }

.input-footer-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-top: 6px;
}

.input-hint-inner {
  font-size: 11px;
  color: #d1d5db;
}

.send-btn {
  width: 34px;
  height: 34px;
  border-radius: 10px;
  border: none;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #e5e7eb;
  color: #9ca3af;
  transition: background 0.2s, color 0.2s, transform 0.1s;
  flex-shrink: 0;
}
.send-btn.ready {
  background: linear-gradient(135deg, #6366f1, #7c3aed);
  color: #fff;
  box-shadow: 0 2px 8px rgba(99,102,241,0.35);
}
.send-btn.ready:hover { filter: brightness(1.08); }
.send-btn.ready:active { transform: scale(0.91); }
.send-btn:disabled { cursor: not-allowed; }

.spin { animation: spin 1s linear infinite; }
@keyframes spin { to { transform: rotate(360deg); } }

.powered-by {
  text-align: center;
  font-size: 11px;
  color: #d1d5db;
  margin: 8px 0 0;
  letter-spacing: 0.02em;
}

/* ── 过渡动画 ── */
.fade-enter-active, .fade-leave-active { transition: opacity 0.25s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }

.msg-enter-active {
  transition: opacity 0.2s ease, transform 0.2s ease;
}
.msg-enter-from {
  opacity: 0;
  transform: translateY(8px);
}

.icon-swap-enter-active, .icon-swap-leave-active {
  transition: opacity 0.15s, transform 0.15s;
}
.icon-swap-enter-from { opacity: 0; transform: scale(0.7); }
.icon-swap-leave-to   { opacity: 0; transform: scale(0.7); }

/* ── 响应式 ── */
@media (max-width: 600px) {
  .messages-inner { padding: 20px 14px 8px; }
  .input-area { padding: 10px 12px 14px; }
  .ai-text, .user-bubble { font-size: 14px; }
  .empty-title { font-size: 18px; }
}
</style>
