<template>
  <main class="app-shell">
    <aside class="sidebar">
      <section class="brand">
        <div class="brand-mark">EM</div>
        <div>
          <h1>YAYO Console</h1>
          <p>Unified debugging for Python and Java versions</p>
        </div>
      </section>

      <a class="profile-card" href="https://xhslink.com/m/558VOQs4Otc" target="_blank" rel="noreferrer">
        <span>My Profile</span>
        <strong>RedNote 69.6K likes and saves</strong>
        <em>View my profile &gt;&gt;</em>
      </a>

      <section class="panel">
        <div class="panel-heading">
          <h2>Backend</h2>
          <span class="pill">{{ currentBackend.label }}</span>
        </div>
        <div class="segmented">
          <button :class="{ active: settings.backend === 'java' }" @click="switchBackend('java')">Java</button>
          <button :class="{ active: settings.backend === 'python' }" @click="switchBackend('python')">Python</button>
        </div>

        <label>
          <span>Java API</span>
          <input v-model="settings.endpoints.java" @change="persist" placeholder="/api/java" />
        </label>
        <label>
          <span>Python API</span>
          <input v-model="settings.endpoints.python" @change="persist" placeholder="/api/python" />
        </label>
        <label>
          <span>User ID</span>
          <input v-model="settings.userId" @change="persist" placeholder="u1001" />
        </label>
        <label>
          <span>Conversation ID</span>
          <input v-model="settings.conversationId" @change="persist" placeholder="Auto-generated" />
        </label>

        <div class="actions">
          <button @click="checkHealth">Health Check</button>
          <button @click="loadStats">Refresh Status</button>
        </div>
      </section>

      <section class="panel status-panel">
        <div class="panel-heading">
          <h2>Status</h2>
          <span :class="['status-dot', healthOk ? 'online' : 'offline']"></span>
        </div>
        <dl>
          <div>
            <dt>Current Backend</dt>
            <dd>{{ currentBackend.label }}</dd>
          </div>
          <div>
            <dt>Health Status</dt>
            <dd :class="healthOk ? 'ok' : 'muted'">{{ healthLabel }}</dd>
          </div>
          <div>
            <dt>Knowledge Chunks</dt>
            <dd>{{ knowledgeCount }}</dd>
          </div>
        </dl>
        <pre v-if="statusText">{{ statusText }}</pre>
      </section>
    </aside>

    <section class="workspace">
      <header class="workspace-header">
        <div>
          <span class="eyebrow">YAYO Workspace</span>
          <h2>Chat Debugging</h2>
          <p>{{ currentBackend.baseUrl }}</p>
        </div>
        <div class="header-actions">
          <a class="profile-link" href="https://xhslink.com/m/558VOQs4Otc" target="_blank" rel="noreferrer">RedNote Profile</a>
          <a :href="docsUrl" target="_blank" rel="noreferrer">API Docs</a>
        </div>
      </header>

      <section class="chat-panel">
        <div class="messages" ref="messageList">
          <article v-for="item in messages" :key="item.id" :class="['message', item.role]">
            <div class="message-meta">
              <span>{{ item.role === 'user' ? 'User' : currentBackend.label }}</span>
              <small v-if="item.meta">{{ item.meta }}</small>
            </div>
            <p>{{ item.content }}</p>
          </article>
          <div v-if="messages.length === 0" class="empty-state">
            <h3>Start a support conversation</h3>
            <p>Switch between Java and Python backends. The frontend automatically adapts response fields.</p>
          </div>
        </div>

        <form class="composer" @submit.prevent="sendMessage">
          <textarea v-model="draft" rows="3" placeholder="Enter a question, for example: I want to request a refund for order #12345"></textarea>
          <button :disabled="busy || !draft.trim()">{{ busy ? 'Sending' : 'Send' }}</button>
        </form>
      </section>

      <section class="tools-grid">
        <article class="tool-panel">
          <div class="panel-heading">
            <h2>Knowledge Search</h2>
            <span class="pill soft">RAG</span>
          </div>
          <div class="inline-form">
            <input v-model="searchQuery" placeholder="How long does a refund take?" />
            <button @click="searchKnowledge" :disabled="busy || !searchQuery.trim()">Search</button>
          </div>
          <div class="result-list">
            <article v-for="item in searchResults" :key="item.id || item.title" class="result-item">
              <strong>{{ item.title || 'Untitled result' }}</strong>
              <span>score {{ item.score ?? '-' }}</span>
              <p>{{ item.content }}</p>
            </article>
          </div>
        </article>

        <article class="tool-panel">
          <div class="panel-heading">
            <h2>Import Knowledge</h2>
            <span class="pill soft">Docs</span>
          </div>
          <label>
            <span>Title</span>
            <input v-model="docTitle" placeholder="Additional refund policy" />
          </label>
          <label>
            <span>Content</span>
            <textarea v-model="docContent" rows="5" placeholder="Enter knowledge base content"></textarea>
          </label>
          <div class="actions">
            <button @click="submitKnowledge" :disabled="busy || !docTitle.trim() || !docContent.trim()">Add Document</button>
            <label class="file-button">
              Upload File
              <input type="file" accept=".txt,.md,.json" @change="handleUpload" />
            </label>
          </div>
        </article>
      </section>
    </section>
  </main>
</template>

<script setup>
import { computed, nextTick, onMounted, reactive, ref, watch } from 'vue'
import {
  addKnowledge,
  backendMeta,
  createInitialSettings,
  requestChat,
  requestHealth,
  requestKnowledgeStats,
  requestMonitor,
  requestSearch,
  saveSettings,
  uploadKnowledge
} from './lib/backends'

const settings = reactive(createInitialSettings())
const messages = ref([])
const draft = ref('')
const busy = ref(false)
const healthOk = ref(false)
const healthLabel = ref('Not checked')
const statusText = ref('')
const knowledgeCount = ref('-')
const searchQuery = ref('How long does a refund take?')
const searchResults = ref([])
const docTitle = ref('Additional refund policy')
const docContent = ref('Refund review time may be extended to 3-5 business days during major promotions.')
const messageList = ref(null)

const currentBackend = computed(() => backendMeta(settings.backend, settings))
const docsUrl = computed(() => {
  if (settings.backend === 'java') return `${currentBackend.value.baseUrl}/docs`
  return `${currentBackend.value.baseUrl}/docs`
})

watch(
  () => settings.conversationId,
  () => persist()
)

onMounted(() => {
  checkHealth()
  loadStats()
})

function switchBackend(type) {
  settings.backend = type
  persist()
  healthOk.value = false
  healthLabel.value = 'Not checked'
  statusText.value = ''
  searchResults.value = []
  checkHealth()
}

function persist() {
  saveSettings(settings)
}

async function sendMessage() {
  const content = draft.value.trim()
  if (!content) return
  messages.value.push({ id: crypto.randomUUID(), role: 'user', content })
  draft.value = ''
  busy.value = true
  try {
    const response = await requestChat(settings.backend, settings, content)
    if (response.conversationId && !settings.conversationId) {
      settings.conversationId = response.conversationId
      persist()
    }
    const meta = [
      response.intent,
      response.agentType,
      response.knowledgeUsed ? 'RAG' : '',
      response.escalated ? 'Escalated to human support' : ''
    ].filter(Boolean).join(' · ')
    messages.value.push({
      id: crypto.randomUUID(),
      role: 'assistant',
      content: response.response,
      meta
    })
  } catch (error) {
    messages.value.push({
      id: crypto.randomUUID(),
      role: 'assistant',
      content: error.message,
      meta: 'Request failed'
    })
  } finally {
    busy.value = false
    await nextTick()
    messageList.value?.scrollTo({ top: messageList.value.scrollHeight, behavior: 'smooth' })
  }
}

async function checkHealth() {
  try {
    const data = await requestHealth(settings.backend, settings)
    healthOk.value = data.status === 'ok'
    healthLabel.value = data.status || 'ok'
    statusText.value = JSON.stringify(data, null, 2)
  } catch (error) {
    healthOk.value = false
    healthLabel.value = 'Unavailable'
    statusText.value = error.message
  }
}

async function loadStats() {
  try {
    const [stats, monitor] = await Promise.allSettled([
      requestKnowledgeStats(settings.backend, settings),
      requestMonitor(settings.backend, settings)
    ])
    if (stats.status === 'fulfilled') {
      knowledgeCount.value = stats.value.total_chunks ?? stats.value.totalChunks ?? '-'
    }
    if (monitor.status === 'fulfilled') {
      statusText.value = JSON.stringify(monitor.value, null, 2)
    }
  } catch (error) {
    statusText.value = error.message
  }
}

async function searchKnowledge() {
  busy.value = true
  try {
    const data = await requestSearch(settings.backend, settings, searchQuery.value, 5)
    searchResults.value = data.results || []
  } catch (error) {
    statusText.value = error.message
  } finally {
    busy.value = false
  }
}

async function submitKnowledge() {
  busy.value = true
  try {
    const data = await addKnowledge(settings.backend, settings, [
      { title: docTitle.value.trim(), content: docContent.value.trim() }
    ])
    statusText.value = JSON.stringify(data, null, 2)
    await loadStats()
  } catch (error) {
    statusText.value = error.message
  } finally {
    busy.value = false
  }
}

async function handleUpload(event) {
  const file = event.target.files?.[0]
  event.target.value = ''
  if (!file) return
  busy.value = true
  try {
    const data = await uploadKnowledge(settings.backend, settings, file)
    statusText.value = JSON.stringify(data, null, 2)
    await loadStats()
  } catch (error) {
    statusText.value = error.message
  } finally {
    busy.value = false
  }
}
</script>
