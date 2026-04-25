<template>
  <div class="chat-window">
    <div class="messages" ref="messagesEl">
      <div
        v-for="(msg, i) in messages"
        :key="i"
        :class="['message', msg.role]"
      >
        <div class="bubble">
          <span class="text">{{ msg.content }}</span>
          <span v-if="msg.streaming" class="cursor" />
        </div>
      </div>
    </div>

    <form class="input-row" @submit.prevent="send">
      <input
        v-model="draft"
        :disabled="streaming"
        placeholder="Type a message..."
        autocomplete="off"
      />
      <button type="submit" :disabled="streaming || !draft.trim()">
        {{ streaming ? '...' : 'Send' }}
      </button>
    </form>
  </div>
</template>

<script setup>
import { ref, nextTick } from 'vue'

const props = defineProps({
  authHeader: { type: String, required: true },
  authToken: { type: String, required: true },
})

const messages = ref([])
const draft = ref('')
const streaming = ref(false)
const messagesEl = ref(null)

const FETCH_OPTS = () => ({
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    [props.authHeader]: props.authToken,
  },
})

async function send() {
  const content = draft.value.trim()
  if (!content || streaming.value) return

  draft.value = ''
  messages.value.push({ role: 'user', content })
  messages.value.push({ role: 'assistant', content: '', streaming: true })
  streaming.value = true
  await scrollToBottom()

  const assistant = messages.value[messages.value.length - 1]

  try {
    // --- streaming attempt ---
    const resp = await fetch('/v1/chat/completions', {
      ...FETCH_OPTS(),
      body: JSON.stringify({ messages: [{ role: 'user', content }], stream: true }),
    })

    if (!resp.ok) {
      const err = await resp.json().catch(() => ({}))
      throw new Error(err?.error?.message ?? `HTTP ${resp.status}`)
    }

    const reader = resp.body.getReader()
    const decoder = new TextDecoder()
    let buffer = ''

    while (true) {
      const { done, value } = await reader.read()
      if (done) break

      buffer += decoder.decode(value, { stream: true })
      const lines = buffer.split('\n')
      buffer = lines.pop()

      for (const line of lines) {
        if (!line.startsWith('data: ')) continue
        const data = line.slice(6).trim()
        if (data === '[DONE]') break
        try {
          const delta = JSON.parse(data).choices?.[0]?.delta?.content ?? ''
          if (delta) {
            assistant.content += delta
            await scrollToBottom()
          }
        } catch { /* ignore malformed chunks */ }
      }
    }

    // nanobot used tools — stream ended before final response
    // fall back to non-streaming to get the complete answer
    if (!assistant.content.trim()) {
      const r = await fetch('/v1/chat/completions', {
        ...FETCH_OPTS(),
        body: JSON.stringify({ messages: [{ role: 'user', content }] }),
      })
      const data = await r.json()
      if (!r.ok) throw new Error(data?.error?.message ?? `HTTP ${r.status}`)
      assistant.content = data.choices?.[0]?.message?.content ?? ''
      await scrollToBottom()
    }
  } catch (e) {
    assistant.content = `Error: ${e.message}`
  } finally {
    assistant.streaming = false
    streaming.value = false
    await scrollToBottom()
  }
}

async function scrollToBottom() {
  await nextTick()
  if (messagesEl.value) {
    messagesEl.value.scrollTop = messagesEl.value.scrollHeight
  }
}
</script>

<style scoped>
.chat-window {
  display: flex;
  flex-direction: column;
  height: 100%;
}

.messages {
  flex: 1;
  overflow-y: auto;
  padding: 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.message {
  display: flex;
}

.message.user {
  justify-content: flex-end;
}

.message.assistant {
  justify-content: flex-start;
}

.bubble {
  max-width: 70%;
  padding: 0.75rem 1rem;
  border-radius: 1rem;
  line-height: 1.5;
  white-space: pre-wrap;
  word-break: break-word;
}

.message.user .bubble {
  background: #2563eb;
  color: #fff;
  border-bottom-right-radius: 0.25rem;
}

.message.assistant .bubble {
  background: #f3f4f6;
  color: #111;
  border-bottom-left-radius: 0.25rem;
}

.cursor {
  display: inline-block;
  width: 2px;
  height: 1em;
  background: currentColor;
  margin-left: 2px;
  vertical-align: text-bottom;
  animation: blink 0.8s step-end infinite;
}

@keyframes blink {
  50% { opacity: 0; }
}

.input-row {
  display: flex;
  gap: 0.5rem;
  padding: 1rem 1.5rem;
  border-top: 1px solid #e5e7eb;
}

.input-row input {
  flex: 1;
  padding: 0.625rem 0.875rem;
  border: 1px solid #d1d5db;
  border-radius: 0.5rem;
  font-size: 1rem;
  outline: none;
  transition: border-color 0.15s;
}

.input-row input:focus {
  border-color: #2563eb;
}

.input-row button {
  padding: 0.625rem 1.25rem;
  background: #2563eb;
  color: #fff;
  border: none;
  border-radius: 0.5rem;
  font-size: 1rem;
  cursor: pointer;
  transition: background 0.15s;
}

.input-row button:hover:not(:disabled) {
  background: #1d4ed8;
}

.input-row button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
</style>
