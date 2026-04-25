<template>
  <div class="app">
    <header>
      <span class="title">nanobot</span>
      <input
        v-if="devMode"
        v-model="authToken"
        class="user-id-input"
        placeholder="User ID"
        title="Sent as X-User-Id header (dev only)"
      />
    </header>
    <ChatWindow :auth-header="authHeader" :auth-token="authToken" />
  </div>
</template>

<script setup>
import { ref } from 'vue'
import ChatWindow from './components/ChatWindow.vue'

const authHeader = import.meta.env.VITE_AUTH_HEADER || 'X-User-Id'
const devMode = authHeader === 'X-User-Id'

// Dev: editable user ID. Prod: static token from env (SSO sets this at build/runtime).
const authToken = ref(import.meta.env.VITE_AUTH_TOKEN || 'dev-user')
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
  max-width: 800px;
  margin: 0 auto;
}

header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1rem 1.5rem;
  border-bottom: 1px solid #e5e7eb;
}

.title {
  font-size: 1.125rem;
  font-weight: 600;
  color: #111;
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
</style>
