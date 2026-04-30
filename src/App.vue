<template>
  <div class="app" :class="{ dark }">
    <div v-if="authMode === 'keycloak' && !ready" class="splash">
      <span v-if="authError" class="splash-error">{{ authError }}</span>
      <span v-else>Signing in…</span>
    </div>

    <template v-else>
      <header>
        <span class="title">nanobot</span>
        <div class="header-right">
          <input
            v-if="devMode"
            v-model="authToken"
            class="user-id-input"
            placeholder="User ID"
            title="Sent as X-User-Id header (dev only)"
          />
          <template v-if="authMode === 'keycloak'">
            <span class="username">{{ userName }}</span>
            <button class="icon-btn" @click="logout" title="Logout">
              <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <path d="M9 21H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h4"/>
                <polyline points="16 17 21 12 16 7"/>
                <line x1="21" y1="12" x2="9" y2="12"/>
              </svg>
            </button>
          </template>
          <button class="icon-btn" @click="toggleTheme" :title="dark ? 'Light mode' : 'Dark mode'">
            <svg v-if="dark" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
              <circle cx="12" cy="12" r="5"/>
              <line x1="12" y1="1" x2="12" y2="3"/>
              <line x1="12" y1="21" x2="12" y2="23"/>
              <line x1="4.22" y1="4.22" x2="5.64" y2="5.64"/>
              <line x1="18.36" y1="18.36" x2="19.78" y2="19.78"/>
              <line x1="1" y1="12" x2="3" y2="12"/>
              <line x1="21" y1="12" x2="23" y2="12"/>
              <line x1="4.22" y1="19.78" x2="5.64" y2="18.36"/>
              <line x1="18.36" y1="5.64" x2="19.78" y2="4.22"/>
            </svg>
            <svg v-else width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
              <path d="M21 12.79A9 9 0 1 1 11.21 3 7 7 0 0 0 21 12.79z"/>
            </svg>
          </button>
        </div>
      </header>
      <div class="chat-wrapper">
        <ChatWindow :auth-header="authHeader" :auth-token="authToken" :storage-key="storageKey" />
      </div>
    </template>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import ChatWindow from './components/ChatWindow.vue'

const authMode = import.meta.env.VITE_AUTH_MODE || 'disabled'
const devMode = authMode === 'disabled'

const authHeader = ref(authMode === 'keycloak' ? 'Authorization' : 'X-User-Id')
const authToken = ref(import.meta.env.VITE_AUTH_TOKEN || 'dev-user')
const userName = ref('')
const storageKey = computed(() => userName.value || authToken.value)
const ready = ref(authMode !== 'keycloak')
const authError = ref('')

// --- theme ---
const dark = ref(localStorage.getItem('theme') !== 'light')

function toggleTheme() {
  dark.value = !dark.value
  localStorage.setItem('theme', dark.value ? 'dark' : 'light')
}

let keycloak = null

function logout() {
  keycloak?.logout({ redirectUri: window.location.origin })
}

onMounted(async () => {
  if (authMode !== 'keycloak') return

  try {
    const Keycloak = (await import('keycloak-js')).default
    keycloak = new Keycloak({
      url: import.meta.env.VITE_KEYCLOAK_URL,
      realm: import.meta.env.VITE_KEYCLOAK_REALM,
      clientId: import.meta.env.VITE_KEYCLOAK_CLIENT_ID,
    })

    const authenticated = await keycloak.init({
      onLoad: 'login-required',
      checkLoginIframe: false,
    })

    if (!authenticated) {
      authError.value = 'Authentication failed.'
      return
    }

    authToken.value = `Bearer ${keycloak.token}`
    userName.value = keycloak.tokenParsed?.preferred_username || keycloak.tokenParsed?.sub || ''

    setInterval(async () => {
      try {
        const refreshed = await keycloak.updateToken(60)
        if (refreshed) authToken.value = `Bearer ${keycloak.token}`
      } catch {
        keycloak.login()
      }
    }, 30_000)

    ready.value = true
  } catch (e) {
    authError.value = `Keycloak init error: ${e.message}`
  }
})
</script>

<style>
/* ---- CSS variables ---- */
:root {
  --bg: #ffffff;
  --bg-sidebar: #f9fafb;
  --bg-hover: #f3f4f6;
  --bg-active: #e8f0fe;
  --bg-input: #ffffff;
  --bg-bubble-assistant: #f3f4f6;
  --border: #e5e7eb;
  --text: #111827;
  --text-muted: #6b7280;
  --text-active: #1d4ed8;
  --accent: #2563eb;
  --accent-hover: #1d4ed8;
  --bubble-user-bg: #2563eb;
  --bubble-user-text: #ffffff;
  --bubble-assistant-bg: #f3f4f6;
  --bubble-assistant-text: #111827;
  --header-bg: #ffffff;
  --input-border: #d1d5db;
  --code-bg: #1e1e1e;
  --code-text: #d4d4d4;
  --inline-code-bg: #e5e7eb;
}

.dark {
  --bg: #1a1b1e;
  --bg-sidebar: #141517;
  --bg-hover: #2c2d30;
  --bg-active: #1e3a5f;
  --bg-input: #2c2d30;
  --bg-bubble-assistant: #2c2d30;
  --border: #2e2f33;
  --text: #e9eaec;
  --text-muted: #868e96;
  --text-active: #74b9ff;
  --accent: #4dabf7;
  --accent-hover: #339af0;
  --bubble-user-bg: #1971c2;
  --bubble-user-text: #ffffff;
  --bubble-assistant-bg: #2c2d30;
  --bubble-assistant-text: #e9eaec;
  --header-bg: #141517;
  --input-border: #2e2f33;
  --code-bg: #0d0e10;
  --code-text: #ced4da;
  --inline-code-bg: #2e2f33;
}

/* ---- Reset ---- */
*, *::before, *::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

html, body, #app {
  height: 100%;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', system-ui, sans-serif;
  background: var(--bg);
  color: var(--text);
  transition: background 0.2s, color 0.2s;
}

/* ---- Layout ---- */
.app {
  display: flex;
  flex-direction: column;
  height: 100%;
}

.chat-wrapper {
  flex: 1;
  min-height: 0;
  overflow: hidden;
  display: flex;
}

/* ---- Header ---- */
header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 1.25rem;
  height: 52px;
  border-bottom: 1px solid var(--border);
  background: var(--header-bg);
  flex-shrink: 0;
  transition: background 0.2s, border-color 0.2s;
}

.title {
  font-size: 1rem;
  font-weight: 700;
  color: var(--text);
  letter-spacing: -0.01em;
}

.header-right {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.username {
  font-size: 0.8125rem;
  color: var(--text-muted);
}

.icon-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 32px;
  height: 32px;
  background: none;
  border: none;
  border-radius: 0.375rem;
  color: var(--text-muted);
  cursor: pointer;
  transition: background 0.15s, color 0.15s;
}

.icon-btn:hover {
  background: var(--bg-hover);
  color: var(--text);
}

.user-id-input {
  padding: 0.375rem 0.75rem;
  border: 1px solid var(--input-border);
  border-radius: 0.375rem;
  font-size: 0.8125rem;
  outline: none;
  background: var(--bg-input);
  color: var(--text-muted);
  width: 140px;
  transition: border-color 0.15s, color 0.15s;
}

.user-id-input:focus {
  border-color: var(--accent);
  color: var(--text);
}

/* ---- Splash ---- */
.splash {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100%;
  color: var(--text-muted);
  font-size: 0.875rem;
}

.splash-error {
  color: #ef4444;
}
</style>
