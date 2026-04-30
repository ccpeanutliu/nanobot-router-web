<template>
  <div class="app">
    <!-- Keycloak: loading / error -->
    <div v-if="authMode === 'keycloak' && !ready" class="splash">
      <span v-if="authError" class="splash-error">{{ authError }}</span>
      <span v-else>Signing in…</span>
    </div>

    <template v-else>
      <header>
        <span class="title">nanobot</span>
        <div class="header-right">
          <!-- Dev mode: editable user id -->
          <input
            v-if="devMode"
            v-model="authToken"
            class="user-id-input"
            placeholder="User ID"
            title="Sent as X-User-Id header (dev only)"
          />
          <!-- Keycloak: show username + logout -->
          <template v-if="authMode === 'keycloak'">
            <span class="username">{{ userName }}</span>
            <button class="logout-btn" @click="logout">Logout</button>
          </template>
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

const authMode = import.meta.env.VITE_AUTH_MODE || 'disabled'  // 'keycloak' | 'disabled'
const devMode = authMode === 'disabled'

// -- disabled mode (dev) --
const authHeader = ref(authMode === 'keycloak' ? 'Authorization' : 'X-User-Id')
const authToken = ref(import.meta.env.VITE_AUTH_TOKEN || 'dev-user')
const userName = ref('')
const storageKey = computed(() => userName.value || authToken.value)
const ready = ref(authMode !== 'keycloak')  // disabled mode is always ready
const authError = ref('')

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
      onLoad: 'login-required',   // redirect to Keycloak if not logged in
      checkLoginIframe: false,
    })

    if (!authenticated) {
      authError.value = 'Authentication failed.'
      return
    }

    authToken.value = `Bearer ${keycloak.token}`
    userName.value = keycloak.tokenParsed?.preferred_username
      || keycloak.tokenParsed?.sub
      || ''

    // Refresh token 60s before expiry
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
*, *::before, *::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

html, body, #app {
  height: 100%;
  font-family: system-ui, sans-serif;
  background: #fff;
}

.app {
  display: flex;
  flex-direction: column;
  height: 100%;
  max-width: 900px;
  margin: 0 auto;
}

.chat-wrapper {
  flex: 1;
  min-height: 0;
  overflow: hidden;
  display: flex;
}

header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1rem 1.5rem;
  border-bottom: 1px solid #e5e7eb;
  flex-shrink: 0;
}

.title {
  font-size: 1.125rem;
  font-weight: 600;
  color: #111;
}

.header-right {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.user-id-input {
  padding: 0.375rem 0.75rem;
  border: 1px solid #d1d5db;
  border-radius: 0.375rem;
  font-size: 0.875rem;
  outline: none;
  color: #6b7280;
  width: 160px;
}

.user-id-input:focus {
  border-color: #2563eb;
  color: #111;
}

.username {
  font-size: 0.875rem;
  color: #6b7280;
}

.logout-btn {
  padding: 0.375rem 0.75rem;
  background: none;
  border: 1px solid #d1d5db;
  border-radius: 0.375rem;
  font-size: 0.875rem;
  color: #374151;
  cursor: pointer;
  transition: background 0.15s;
}

.logout-btn:hover {
  background: #f3f4f6;
}

.splash {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100%;
  color: #6b7280;
  font-size: 0.875rem;
}

.splash-error {
  color: #ef4444;
}
</style>
