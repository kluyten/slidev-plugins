<template>
  <div class="java-playground" :class="{ 'compact': compact }">
    <!-- Use iframe mode if embedUrl is provided -->
    <div v-if="embedUrl" class="runner-frame">
      <iframe
        :src="embedUrl"
        title="Java Playground"
        frameborder="0"
        allow="clipboard-write; fullscreen"
        width="100%"
        :height="editorHeight"
        referrerpolicy="no-referrer-when-downgrade"
      />
    </div>

    <!-- Otherwise use built-in editor with API -->
    <div v-else class="playground-container">
      <div class="playground-header">
        <div class="header-left">
          <span class="playground-title">☕ Java Playground</span>
          <span v-if="fileName" class="file-name">{{ fileName }}</span>
        </div>
        <div class="header-right">
          <button
            @click="runCode"
            :disabled="loading"
            class="run-button"
          >
            <span v-if="!loading">▶ Run</span>
            <span v-else>⟳ Running...</span>
          </button>
          <button
            @click="resetCode"
            class="reset-button"
            title="Reset to original code"
          >
            ↻ Reset
          </button>
        </div>
      </div>

      <div class="playground-body" :style="{ height: computedHeight }">
        <div class="editor-section" :style="{ flex: showOutput ? '0 0 60%' : '1' }">
          <textarea
            v-model="currentCode"
            class="code-editor"
            spellcheck="false"
            @keydown.tab.prevent="handleTab"
          ></textarea>
        </div>

        <div v-if="showOutput" class="output-section">
          <div class="output-header">
            <span>Output</span>
            <button @click="clearOutput" class="clear-button" title="Clear output">×</button>
          </div>
          <pre class="output-content" :class="{ 'error': hasError }">{{ output || 'Click Run to see output...' }}</pre>
        </div>
      </div>

      <div v-if="showStats && executionTime" class="stats-bar">
        ⏱️ Execution time: {{ executionTime }}ms | Memory: {{ memoryUsage }}KB
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import axios from 'axios'

/**
 * JavaPlayground.vue - Interactive Java code editor
 *
 * Two modes:
 * 1. Iframe mode: Use JDoodle embed URL (no API keys needed)
 * 2. Editor mode: Built-in editor with JDoodle API (requires API keys)
 *
 * Props:
 * - embedUrl: JDoodle embed URL (optional, enables iframe mode)
 * - code: Initial Java code (for editor mode)
 * - fileName: Optional file name to display
 * - editorHeight: Height of editor (default: '500px')
 * - compact: Compact mode (default: false)
 * - showOutput: Show output panel (default: true)
 * - showStats: Show execution stats (default: false)
 */

const props = withDefaults(defineProps<{
  embedUrl?: string
  code?: string
  fileName?: string
  editorHeight?: string
  compact?: boolean
  showOutput?: boolean
  showStats?: boolean
}>(), {
  code: '// Write your Java code here\npublic class Main {\n    public static void main(String[] args) {\n        System.out.println("Hello, Java!");\n    }\n}',
  editorHeight: '500px',
  compact: false,
  showOutput: true,
  showStats: false
})

const currentCode = ref(props.code)
const originalCode = ref(props.code)
const output = ref('')
const loading = ref(false)
const hasError = ref(false)
const executionTime = ref(0)
const memoryUsage = ref(0)

const computedHeight = computed(() => {
  return props.editorHeight.replace(/px$/, '') + 'px'
})

// JDoodle API configuration - use environment variables
// Use Vite proxy to bypass CORS: /api/jdoodle/v1/execute -> https://api.jdoodle.com/v1/execute
const isDev = window.location.hostname === 'localhost' || window.location.hostname === '127.0.0.1'
const JDOODLE_API_URL = isDev
  ? '/api/jdoodle/v1/execute'  // Use proxy in development to bypass CORS
  : 'https://api.jdoodle.com/v1/execute'  // Direct in production

// @ts-ignore - Vite injects env variables
const JDOODLE_CLIENT_ID = import.meta.env?.VITE_JDOODLE_CLIENT_ID || ''
// @ts-ignore - Vite injects env variables
const JDOODLE_CLIENT_SECRET = import.meta.env?.VITE_JDOODLE_CLIENT_SECRET || ''

const runCode = async () => {
  if (!JDOODLE_CLIENT_ID || !JDOODLE_CLIENT_SECRET) {
    hasError.value = true
    output.value = '⚠️ API Configuration Required:\n\nCreate a .env file with:\nVITE_JDOODLE_CLIENT_ID=your-client-id\nVITE_JDOODLE_CLIENT_SECRET=your-secret\n\nGet free API keys at: https://www.jdoodle.com/compiler-api\n(Free tier: 200 requests/day)\n\nAlternatively, use the embedUrl prop with a JDoodle embed link for unlimited usage.'
    return
  }

  loading.value = true
  hasError.value = false
  output.value = 'Compiling and running...'

  try {
    const startTime = performance.now()

    const response = await axios.post(JDOODLE_API_URL, {
      clientId: JDOODLE_CLIENT_ID,
      clientSecret: JDOODLE_CLIENT_SECRET,
      script: currentCode.value,
      language: 'java',
      versionIndex: '4', // Java 17
    }, {
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json'
      },
      timeout: 30000 // 30 second timeout
    })

    const endTime = performance.now()
    executionTime.value = Math.round(endTime - startTime)

    if (response.data) {
      const result = response.data

      if (result.statusCode === 200) {
        output.value = result.output || '(No output)'
        memoryUsage.value = result.memory || 0
        hasError.value = false
      } else {
        output.value = result.output || result.error || 'Execution failed'
        hasError.value = true
      }
    }
  } catch (error: any) {
    hasError.value = true
    console.error('JDoodle API Error:', error)

    if (error.code === 'ERR_NETWORK' || error.message === 'Network Error') {
      output.value = '⚠️ Network Error - CORS or Connectivity Issue:\n\n' +
        'Common causes:\n' +
        '1. Browser CORS restrictions (try Firefox or disable CORS in dev)\n' +
        '2. JDoodle API may block requests from localhost\n' +
        '3. Invalid API credentials\n' +
        '4. No internet connection\n\n' +
        'Solution: Use embedUrl prop with JDoodle embed for presentations:\n' +
        '<JavaPlayGround embedUrl="https://www.jdoodle.com/embed/v0/..." />\n\n' +
        'Or install a CORS browser extension for development.'
    } else if (error.response?.status === 401) {
      output.value = '⚠️ API Authentication Failed:\n\nCheck your JDoodle credentials in .env file.\nMake sure they are correct.'
    } else if (error.response?.status === 429) {
      output.value = '⚠️ API Rate Limit Exceeded:\n\nFree tier: 200 requests/day.\nTry again later or upgrade at jdoodle.com'
    } else if (error.code === 'ECONNABORTED') {
      output.value = '⚠️ Request Timeout:\n\nThe code took too long to execute.\nSimplify your code or check your connection.'
    } else {
      output.value = `Error: ${error.message}\n\nDetails: ${JSON.stringify(error.response?.data || {}, null, 2)}`
    }
  } finally {
    loading.value = false
  }
}

const resetCode = () => {
  currentCode.value = originalCode.value
  output.value = ''
  hasError.value = false
}

const clearOutput = () => {
  output.value = ''
  hasError.value = false
}

const handleTab = (e: KeyboardEvent) => {
  const target = e.target as HTMLTextAreaElement
  const start = target.selectionStart
  const end = target.selectionEnd

  currentCode.value =
    currentCode.value.substring(0, start) +
    '    ' +
    currentCode.value.substring(end)

  setTimeout(() => {
    target.selectionStart = target.selectionEnd = start + 4
  }, 0)
}

onMounted(() => {
  if (props.code) {
    originalCode.value = props.code
    currentCode.value = props.code
  }
})
</script>

<style scoped>
.java-playground {
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  font-family: 'SF Mono', Monaco, 'Cascadia Code', 'Roboto Mono', Consolas, 'Courier New', monospace;
}

.runner-frame {
  width: 100%;
  max-width: 1100px;
  margin: 0 auto;
}

.playground-container {
  background: #1e1e1e;
}

.java-playground.compact {
  font-size: 0.85em;
}

.playground-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 16px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
}

.header-left {
  display: flex;
  align-items: center;
  gap: 12px;
}

.playground-title {
  font-weight: 600;
  font-size: 0.95em;
}

.file-name {
  font-size: 0.85em;
  opacity: 0.9;
  background: rgba(255, 255, 255, 0.2);
  padding: 2px 8px;
  border-radius: 4px;
}

.header-right {
  display: flex;
  gap: 8px;
}

.run-button,
.reset-button {
  padding: 6px 16px;
  border: none;
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s ease;
  font-size: 0.9em;
}

.run-button {
  background: #48bb78;
  color: white;
}

.run-button:hover:not(:disabled) {
  background: #38a169;
  transform: translateY(-1px);
}

.run-button:disabled {
  background: #4a5568;
  cursor: not-allowed;
}

.reset-button {
  background: rgba(255, 255, 255, 0.2);
  color: white;
}

.reset-button:hover {
  background: rgba(255, 255, 255, 0.3);
}

.playground-body {
  display: flex;
  background: #1e1e1e;
}

.editor-section {
  display: flex;
  flex-direction: column;
  border-right: 1px solid #333;
}

.code-editor {
  flex: 1;
  width: 100%;
  padding: 12px;
  background: #1e1e1e;
  color: #d4d4d4;
  border: none;
  resize: none;
  font-family: inherit;
  font-size: 0.6em;
  line-height: 1.5;
  outline: none;
  tab-size: 4;
}

.code-editor:focus {
  background: #252526;
}

.output-section {
  flex: 0 0 40%;
  display: flex;
  flex-direction: column;
  background: #2d2d2d;
}

.output-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 16px;
  background: #252526;
  color: #d4d4d4;
  font-size: 0.85em;
  font-weight: 600;
  border-bottom: 1px solid #333;
}

.clear-button {
  background: none;
  border: none;
  color: #d4d4d4;
  font-size: 1.5em;
  cursor: pointer;
  padding: 0 4px;
  opacity: 0.6;
  transition: opacity 0.2s;
}

.clear-button:hover {
  opacity: 1;
}

.output-content {
  flex: 1;
  padding: 12px;
  margin: 0;
  color: #d4d4d4;
  font-family: inherit;
  font-size: 0.75em;
  line-height: 1.4;
  overflow: auto;
  white-space: pre-wrap;
  word-wrap: break-word;
}

.output-content.error {
  color: #f56565;
}

.stats-bar {
  padding: 8px 16px;
  background: #252526;
  color: #9ca3af;
  font-size: 0.8em;
  border-top: 1px solid #333;
}

/* Scrollbar styling */
.code-editor::-webkit-scrollbar,
.output-content::-webkit-scrollbar {
  width: 8px;
  height: 8px;
}

.code-editor::-webkit-scrollbar-track,
.output-content::-webkit-scrollbar-track {
  background: #1e1e1e;
}

.code-editor::-webkit-scrollbar-thumb,
.output-content::-webkit-scrollbar-thumb {
  background: #4a5568;
  border-radius: 4px;
}

.code-editor::-webkit-scrollbar-thumb:hover,
.output-content::-webkit-scrollbar-thumb:hover {
  background: #5a6678;
}
</style>