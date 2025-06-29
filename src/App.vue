<template>
  <div class="app-container">
    <div class="container">
      <div class="position-relative">
        <h4 class="text-center mb-3">
          <i class="fas fa-robot me-2"></i>
          AI Chat Assistant
        </h4>

        <div :class="['connection-status', isOnline ? 'status-online' : 'status-offline']">
          <i :class="['fas', isOnline ? 'fa-circle' : 'fa-circle-xmark']"></i>
          <span class="d-none d-sm-inline">{{ isOnline ? ' Online' : ' Offline' }}</span>
        </div>
      </div>

      <div class="chat-container" ref="chatContainer">
        <div v-if="messages.length === 0" class="empty-state text-center py-5">
          <i class="fas fa-comments fa-3x text-muted mb-3"></i>
          <p class="text-muted">Start a conversation with your AI assistant</p>
        </div>
        
        <div v-for="(msg, index) in messages" :key="index" class="message-wrapper">
          <div v-if="msg.sender === 'user'" :class="['message-bubble', 'user-message', 'text-white']">
            <div class="message-text">{{ msg.text }}</div>
            <div class="timestamp text-end">{{ formatTime(msg.timestamp) }}</div>
          </div>

          <div v-else-if="msg.sender === 'bot'" :class="['message-bubble', 'bot-message', 'text-white']">
            <!-- Text content -->
            <div v-if="msg.type === 'text'" class="message-text">{{ msg.text }}</div>

            <!-- Code block with enhanced features -->
            <div v-if="msg.type === 'code'" class="code-message">
              <div v-if="msg.text" class="message-text mb-2">{{ msg.text }}</div>
              <div class="code-container">
                <div class="code-header">
                  <span class="code-language">{{ getLanguageDisplayName(msg.language || 'text') }}</span>
                  <button 
                    @click="copyCode(msg.code, msg.timestamp)" 
                    class="copy-button"
                    :class="{ 'copied': copiedStates[msg.timestamp] }"
                    :title="copiedStates[msg.timestamp] ? 'Copied!' : 'Copy code'"
                  >
                    <i :class="copiedStates[msg.timestamp] ? 'fas fa-check' : 'fas fa-copy'"></i>
                  </button>
                </div>
                <pre class="code-block"><code :class="`language-${msg.language || 'text'}`">{{ msg.code }}</code></pre>
              </div>
            </div>

            <!-- Mixed content (text + code blocks) -->
            <div v-if="msg.type === 'mixed'" class="mixed-message">
              <div v-if="msg.text" class="message-text mb-2">{{ msg.text }}</div>
              <div v-for="(block, blockIndex) in msg.codeBlocks" :key="blockIndex" class="code-container mb-2">
                <div class="code-header">
                  <span class="code-language">{{ getLanguageDisplayName(block.language) }}</span>
                  <button 
                    @click="copyCode(block.code, `${msg.timestamp}-${blockIndex}`)" 
                    class="copy-button"
                    :class="{ 'copied': copiedStates[`${msg.timestamp}-${blockIndex}`] }"
                    :title="copiedStates[`${msg.timestamp}-${blockIndex}`] ? 'Copied!' : 'Copy code'"
                  >
                    <i :class="copiedStates[`${msg.timestamp}-${blockIndex}`] ? 'fas fa-check' : 'fas fa-copy'"></i>
                  </button>
                </div>
                <pre class="code-block"><code :class="`language-${block.language}`">{{ block.code }}</code></pre>
              </div>
            </div>

            <!-- Image content -->
            <div v-if="msg.type === 'image' || msg.image">
              <div v-if="msg.text" class="message-text mb-2">{{ msg.text }}</div>
              <div class="media-content">
                <img :src="msg.image || msg.url" alt="AI generated image" class="img-fluid" loading="lazy" />
              </div>
            </div>

            <!-- Video content -->
            <div v-if="msg.type === 'video' || msg.video">
              <div v-if="msg.text" class="message-text mb-2">{{ msg.text }}</div>
              <div class="media-content">
                <video controls class="w-100" preload="metadata">
                  <source :src="msg.video || msg.url" type="video/mp4" />
                  Your browser doesn't support video playbook.
                </video>
              </div>
            </div>

            <!-- Audio content -->
            <div v-if="msg.type === 'audio' || msg.audio">
              <div v-if="msg.text" class="message-text mb-2">{{ msg.text }}</div>
              <audio controls class="audio-player" preload="metadata">
                <source :src="msg.audio || msg.url" type="audio/mpeg" />
                Your browser doesn't support audio playback.
              </audio>
            </div>

            <!-- File content -->
            <div v-if="msg.type === 'file' || msg.file">
              <div v-if="msg.text" class="message-text mb-2">{{ msg.text }}</div>
              <div class="file-download">
                <i class="fas fa-file-alt"></i>
                <a :href="msg.file || msg.url" target="_blank" class="text-decoration-none">
                  {{ msg.filename || 'Download File' }}
                </a>
              </div>
            </div>

            <div class="timestamp">{{ formatTime(msg.timestamp) }}</div>
          </div>

          <div v-else-if="msg.type === 'error'" class="error-message">
            <i class="fas fa-exclamation-triangle me-2"></i>
            {{ msg.text }}
          </div>
        </div>

        <div v-if="isTyping" class="typing-indicator">
          <div class="typing-dots">
            <div class="typing-dot"></div>
            <div class="typing-dot"></div>
            <div class="typing-dot"></div>
          </div>
          <span class="typing-text">AI is typing...</span>
        </div>
      </div>

      <!-- Scroll to bottom button -->
      <div v-if="!isNearBottom() && messages.length > 0" class="scroll-to-bottom-btn" @click="scrollToBottomSmooth">
        <i class="fas fa-chevron-down"></i>
      </div>

      <div class="input-section">
        <div class="input-wrapper d-flex align-items-center">
          <input
            v-model="prompt"
            @keydown.enter="sendPrompt"
            type="text"
            class="form-control message-input"
            placeholder="Type your message..."
            :disabled="isTyping"
            autocomplete="off"
            spellcheck="true"
          />
          <button
            class="send-button btn btn-primary"
            @click="sendPrompt"
            :disabled="!prompt.trim() || isTyping"
            :title="isTyping ? 'Please wait...' : 'Send message'"
            type="button"
          >
            <i v-if="isTyping" class="fas fa-spinner fa-spin"></i>
            <i v-else class="fas fa-paper-plane"></i>
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, nextTick, onMounted, onUnmounted } from 'vue'

// Reactive data
const prompt = ref('')
const messages = ref([])
const chatContainer = ref(null)
const isTyping = ref(false)
const isOnline = ref(true)
const copiedStates = ref({})
let connectionCheckInterval = null

// Language display names mapping
const languageDisplayNames = {
  javascript: 'JavaScript',
  typescript: 'TypeScript',
  python: 'Python',
  java: 'Java',
  cpp: 'C++',
  html: 'HTML',
  css: 'CSS',
  sql: 'SQL',
  json: 'JSON',
  xml: 'XML',
  bash: 'Bash',
  php: 'PHP',
  ruby: 'Ruby',
  go: 'Go',
  rust: 'Rust',
  swift: 'Swift',
  kotlin: 'Kotlin',
  text: 'Text'
}

// Enhanced code detection functions
const detectCodeLanguage = (code) => {
  const patterns = {
    javascript: /(?:function|const|let|var|=>|console\.log|require|import)/i,
    python: /(?:def |import |from |print\(|if __name__|class )/i,
    java: /(?:public class|public static void|import java|System\.out\.println)/i,
    cpp: /(?:#include|using namespace|int main\(|std::)/i,
    html: /(?:<html|<div|<span|<p>|<!DOCTYPE)/i,
    css: /(?:\{[^}]*[a-z-]+\s*:\s*[^}]+\}|@media|\.[\w-]+\s*\{)/i,
    sql: /(?:SELECT|INSERT|UPDATE|DELETE|CREATE TABLE|ALTER TABLE)/i,
    json: /^\s*[\{\[][\s\S]*[\}\]]\s*$/,
    xml: /^\s*<\?xml|<[a-zA-Z][\s\S]*>/,
    bash: /(?:#!\/bin\/bash|ls |cd |mkdir |chmod |sudo )/i,
    php: /(?:<\?php|\$[a-zA-Z_]|function |class |echo )/i,
    ruby: /(?:def |class |require |puts |end$)/i,
    go: /(?:package |import |func |fmt\.)/i,
    rust: /(?:fn |let |use |struct |impl )/i,
    swift: /(?:func |var |let |import |class |struct )/i,
    kotlin: /(?:fun |val |var |class |package )/i,
    typescript: /(?:interface |type |enum |namespace |declare )/i
  }

  for (const [lang, pattern] of Object.entries(patterns)) {
    if (pattern.test(code)) {
      return lang
    }
  }
  return 'text'
}

const extractCodeBlocks = (text) => {
  const codeBlocks = []
  let remainingText = text
  
  // Match fenced code blocks (```language\ncode\n```)
  const fencedRegex = /```(\w+)?\n?([\s\S]*?)```/g
  let match
  
  while ((match = fencedRegex.exec(text)) !== null) {
    const language = match[1] || detectCodeLanguage(match[2])
    codeBlocks.push({
      type: 'code',
      language: language,
      code: match[2].trim(),
      fullMatch: match[0]
    })
  }
  
  // Remove code blocks from remaining text
  remainingText = text.replace(fencedRegex, '').trim()
  
  // Check for inline code blocks without fences
  if (codeBlocks.length === 0) {
    const detectedLang = detectCodeLanguage(text)
    if (detectedLang !== 'text') {
      codeBlocks.push({
        type: 'code',
        language: detectedLang,
        code: text.trim(),
        fullMatch: text
      })
      remainingText = ''
    }
  }
  
  return {
    codeBlocks,
    remainingText
  }
}

const scrollToBottomSmooth = () => {
  nextTick(() => {
    if (chatContainer.value) {
      requestAnimationFrame(() => {
        chatContainer.value.scrollTo({
          top: chatContainer.value.scrollHeight,
          behavior: 'smooth'
        })
      })
    }
  })
}

const isNearBottom = () => {
  if (!chatContainer.value) return true
  const { scrollTop, scrollHeight, clientHeight } = chatContainer.value
  const threshold = 100
  return scrollHeight - scrollTop - clientHeight < threshold
}

const formatTime = (timestamp) => {
  if (!timestamp) return ''
  return new Date(timestamp).toLocaleTimeString([], { 
    hour: '2-digit', 
    minute: '2-digit' 
  })
}

const cleanText = (text) => {
  if (!text) return ''
  return text
    .replace(/([.!?])([A-Za-z])/g, '$1 $2 ')
    .replace(/\s+/g, ' ')
    .replace(/([A-Za-z])([A-Z])/g, '$1 $2 ')
    .trim()
}

const detectContentType = (content) => {
  if (!content) return 'text'
  
  if (content.includes('data:image/') || content.startsWith('image:')) {
    return 'image'
  }
  
  if (content.includes('data:video/') || content.match(/\.(mp4|webm|ogg)$/i)) {
    return 'video'
  }
  
  if (content.includes('data:audio/') || content.match(/\.(mp3|wav|ogg|m4a)$/i)) {
    return 'audio'
  }
  
  if (content.match(/\.(pdf|doc|docx|txt|zip|rar)$/i)) {
    return 'file'
  }
  
  if (content.includes('```') || content.startsWith('code:')) {
    return 'code'
  }
  
  return 'text'
}

const getLanguageDisplayName = (language) => {
  return languageDisplayNames[language] || language.charAt(0).toUpperCase() + language.slice(1)
}

// Copy to clipboard function
const copyToClipboard = async (text) => {
  try {
    await navigator.clipboard.writeText(text)
    return true
  } catch (err) {
    const textArea = document.createElement('textarea')
    textArea.value = text
    document.body.appendChild(textArea)
    textArea.select()
    document.execCommand('copy')
    document.body.removeChild(textArea)
    return true
  }
}

const copyCode = async (code, key) => {
  const success = await copyToClipboard(code)
  if (success) {
    copiedStates.value[key] = true
    setTimeout(() => {
      delete copiedStates.value[key]
    }, 2000)
  }
}

// Enhanced AI response processing
const processAIResponse = (jsonResponse) => {
  const response = jsonResponse.response?.trim()
  if (!response) return null

  const message = {
    sender: 'bot',
    timestamp: Date.now(),
    text: '',
    type: 'text'
  }

  // Case 1: Direct base64 image in response
  if (jsonResponse.image_bytes) {
    message.type = 'image'
    message.image = `data:image/png;base64,${jsonResponse.image_bytes.trim()}`
    return message
  }

  // Case 2: Image prefix in response
  if (response.startsWith('image:')) {
    message.type = 'image'
    message.image = `data:image/png;base64,${response.replace('image:', '').trim()}`
    return message
  }

  // Case 3: Video prefix in response
  if (response.startsWith('video:')) {
    message.type = 'video'
    message.video = response.replace('video:', '').trim()
    return message
  }

  // Case 4: Audio prefix in response
  if (response.startsWith('audio:')) {
    message.type = 'audio'
    message.audio = response.replace('audio:', '').trim()
    return message
  }

  // Case 5: File prefix in response
  if (response.startsWith('file:')) {
    message.type = 'file'
    const fileData = response.replace('file:', '').trim()
    const parts = fileData.split('|')
    message.file = parts[0]
    message.filename = parts[1] || 'Download File'
    return message
  }

  // Case 6: Code prefix in response
  if (response.startsWith('code:')) {
    message.type = 'code'
    const codeContent = response.replace('code:', '').trim()
    message.code = codeContent
    message.language = detectCodeLanguage(codeContent)
    return message
  }

  // Enhanced Case 7: Code blocks detection
  const { codeBlocks, remainingText } = extractCodeBlocks(response)
  
  if (codeBlocks.length > 0) {
    // If there's only one code block and no other text, return as code message
    if (codeBlocks.length === 1 && !remainingText) {
      message.type = 'code'
      message.code = codeBlocks[0].code
      message.language = codeBlocks[0].language
      return message
    }
    
    // If there are multiple code blocks or mixed content, return as mixed message
    message.type = 'mixed'
    message.text = remainingText
    message.codeBlocks = codeBlocks
    return message
  }

  // Case 8: URL detection
  const urlMatch = response.match(/(https?:\/\/[^\s]+)/)
  if (urlMatch) {
    const url = urlMatch[1]
    const contentType = detectContentType(url)
    
    if (contentType !== 'text') {
      message.type = contentType
      message[contentType] = url
      message.text = response.replace(url, '').trim()
      return message
    }
  }

  // Case 9: Pure code detection (no fences)
  const detectedLanguage = detectCodeLanguage(response)
  if (detectedLanguage !== 'text' && response.split('\n').length > 2) {
    message.type = 'code'
    message.code = response
    message.language = detectedLanguage
    return message
  }

  // Default: text response
  message.text = cleanText(response)
  return message
}

const sendPrompt = async () => {
  if (!prompt.value.trim() || isTyping.value) return

  const userPrompt = prompt.value
  messages.value.push({ 
    sender: 'user', 
    type: 'text', 
    text: userPrompt,
    timestamp: Date.now()
  })
  prompt.value = ''
  scrollToBottomSmooth()

  isTyping.value = true
  let completeResponse = ''

  try {
    const response = await fetch('http://localhost:11434/api/generate', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        model: 'phi3',
        prompt: userPrompt,
        stream: true
      })
    })

    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`)
    }

    isOnline.value = true
    const reader = response.body.getReader()
    const decoder = new TextDecoder('utf-8')
    let buffer = ''

    while (true) {
      const { value, done } = await reader.read()
      if (done) break

      buffer += decoder.decode(value, { stream: true })

      let newlineIndex
      while ((newlineIndex = buffer.indexOf('\n')) !== -1) {
        const line = buffer.slice(0, newlineIndex).trim()
        buffer = buffer.slice(newlineIndex + 1)

        if (!line) continue

        try {
          const json = JSON.parse(line)
          
          if (json.response !== undefined) {
            completeResponse += json.response
          } else {
            const processedMessage = processAIResponse(json)
            
            if (processedMessage && processedMessage.type === 'text') {
              completeResponse += processedMessage.text
            }
          }
        } catch (err) {
          console.warn('JSON parsing error:', err.message)
        }
      }
    }

    // Process the complete response
    if (completeResponse.trim()) {
      const processedMessage = processAIResponse({ response: completeResponse.trim() })
      if (processedMessage) {
        messages.value.push(processedMessage)
      }
    }

  } catch (err) {
    console.error('Fetch error:', err)
    isOnline.value = false
    messages.value.push({
      type: 'error',
      text: `Connection failed: ${err.message}. Please check if Ollama is running on localhost:11434`,
      timestamp: Date.now()
    })
  } finally {
    isTyping.value = false
    scrollToBottomSmooth()
  }
}

// Connection check function
const checkConnection = async () => {
  try {
    const controller = new AbortController()
    const timeoutId = setTimeout(() => controller.abort(), 5000)
    
    const response = await fetch('http://localhost:11434/api/tags', {
      method: 'GET',
      signal: controller.signal
    })
    
    clearTimeout(timeoutId)
    isOnline.value = response.ok
  } catch {
    isOnline.value = false
  }
}

// Lifecycle hooks
onMounted(() => {
  checkConnection()
  connectionCheckInterval = setInterval(checkConnection, 30000)
  
  if (chatContainer.value) {
    chatContainer.value.addEventListener('scroll', () => {
      nextTick(() => {})
    })
  }
})

onUnmounted(() => {
  if (connectionCheckInterval) {
    clearInterval(connectionCheckInterval)
  }
  
  if (chatContainer.value) {
    chatContainer.value.removeEventListener('scroll', () => {})
  }
})
</script>

<style scoped>
.app-container {
  height: 90vh;
  overflow: hidden;
  margin-top: 40px;
  margin-bottom: 0px;
}

.chat-container {
  height: 75vh;
  min-height: 400px;
  max-height: 600px;
  padding: 16px;
  overflow-y: auto;
  scroll-behavior: smooth;
  margin-bottom: 0px;
}

.message-bubble {
  max-width: 85%;
  word-wrap: break-word;
  border-radius: 18px;
  position: relative;
  animation: slideIn 0.3s ease-out;
  padding: 8px;
  margin-top: 16px;
}

.user-message {
  background: linear-gradient(135deg, #007bff, #0056b3);
  margin-left: auto;
  border-bottom-right-radius: 8px;
}

.bot-message {
  background: linear-gradient(135deg, #6c757d, #495057);
  margin-right: auto;
  border-bottom-left-radius: 8px;
}

.message-text {
  white-space: pre-wrap;
  word-break: break-word;
  padding: 8px;
  margin-top: 16px;
}

/* Enhanced Code Styling */
.code-message {
  margin-top: 8px;
}

.code-container {
  background: #1a1a1a;
  border-radius: 8px;
  overflow: hidden;
  margin-top: 8px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.code-header {
  background: #2d3748;
  color: #e2e8f0;
  padding: 8px 12px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid #4a5568;
}

.code-language {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  color: #9ca3af;
}

.copy-button {
  background: transparent;
  border: none;
  color: #9ca3af;
  cursor: pointer;
  padding: 4px 8px;
  border-radius: 4px;
  transition: all 0.2s;
  font-size: 0.8rem;
}

.copy-button:hover {
  background: #4a5568;
  color: #e2e8f0;
}

.copy-button.copied {
  color: #10b981;
}

.code-block {
  background: #1a1a1a;
  color: #e2e8f0;
  padding: 12px;
  margin: 0;
  overflow-x: auto;
  font-family: 'Fira Code', 'Courier New', monospace;
  font-size: 0.85rem;
  line-height: 1.4;
  white-space: pre;
  border: none;
}

.code-block code {
  background: none;
  padding: 0;
  font-size: inherit;
  color: inherit;
}

.mixed-message {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

/* Syntax highlighting colors */
.language-javascript .keyword { color: #ff79c6; }
.language-javascript .string { color: #f1fa8c; }
.language-javascript .function { color: #50fa7b; }
.language-javascript .comment { color: #6272a4; }

.language-python .keyword { color: #ff79c6; }
.language-python .string { color: #f1fa8c; }
.language-python .function { color: #50fa7b; }
.language-python .comment { color: #6272a4; }

.language-html .tag { color: #ff79c6; }
.language-html .attr-name { color: #50fa7b; }
.language-html .attr-value { color: #f1fa8c; }

.language-css .property { color: #50fa7b; }
.language-css .value { color: #f1fa8c; }
.language-css .selector { color: #ff79c6; }

.typing-indicator {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 12px 16px;
  background: #f8f9fa;
  border-radius: 18px;
  border-bottom-left-radius: 8px;
  max-width: 85%;
  margin-right: auto;
}

.typing-dots {
  display: flex;
  gap: 4px;
}

.typing-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: #6c757d;
  animation: typing 1.4s infinite ease-in-out;
}

.typing-dot:nth-child(1) { animation-delay: -0.32s; }
.typing-dot:nth-child(2) { animation-delay: -0.16s; }

@keyframes typing {
  0%, 80%, 100% { transform: scale(0.8); opacity: 0.5; }
  40% { transform: scale(1); opacity: 1; }
}

@keyframes slideIn {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}

.media-content {
  border-radius: 12px;
  overflow: hidden;
  margin-top: 8px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.media-content img {
  max-width: 100%;
  height: auto;
  display: block;
}

.media-content video {
  width: 100%;
  max-height: 300px;
}

.audio-player {
  width: 100%;
  margin-top: 8px;
}

.file-download {
  background: #f8f9fa;
  border: 1px solid #dee2e6;
  border-radius: 8px;
  padding: 12px;
  margin-top: 8px;
  display: flex;
  align-items: center;
  gap: 8px;
}

.file-download a {
  color: #007bff;
  font-weight: 500;
}

.input-section {
  background: white;
  border-top: 1px solid #e9ecef;
  padding: 12px;
  position: sticky;
  bottom: 0;
  width: 100%;
}

.send-button {
  border-radius: 50%;
  width: 48px;
  height: 48px;
  display: flex;
  margin-left: 10px;
  align-items: center;
  justify-content: center;
  transition: all 0.2s;
}

.send-button:hover {
  transform: scale(1.05);
}

.send-button:disabled {
  opacity: 0.6;
  transform: none;
}

.message-input {
  border-radius: 24px;
  border: 2px solid #e9ecef;
  padding: 12px 20px;
  font-size: 16px;
  transition: border-color 0.2s;
}

.message-input:focus {
  border-color: #007bff;
  box-shadow: 0 0 0 0.2rem rgba(0,123,255,.25);
}

.error-message {
  background: #f8d7da;
  color: #721c24;
  border: 1px solid #f5c6cb;
  border-radius: 12px;
  padding: 12px;
  margin: 8px 0;
}

.timestamp {
  font-size: 0.75rem;
  opacity: 0.7;
  margin-top: 4px;
}

.connection-status {
  position: absolute;
  top: 10px;
  right: 10px;
  padding: 4px 8px;
  border-radius: 12px;
  font-size: 0.75rem;
  font-weight: 500;
}

.connection-status {
  position: absolute;
  top: 10px;
  right: 10px;
  padding: 4px 8px;
  border-radius: 12px;
  font-size: 0.75rem;
  font-weight: 500;
}

.status-online { background: #d4edda; color: #155724; }
.status-offline { background: #f8d7da; color: #721c24; }

/* Mobile optimizations */
@media (max-width: 768px) {
  .chat-container {
    height: calc(100vh - 160px);
  }
  
  .message-bubble {
    max-width: 95%;
  }
  
  .send-button {
    width: 44px;
    height: 44px;
  }
  
  .message-input {
    font-size: 16px;
  }
}

@media (max-width: 480px) {
  .message-bubble {
    max-width: 98%;
  }
  
  .container-fluid {
    padding-top: 8px;
    padding-left: 8px;
    padding-right: 8px;
  }
}
</style>