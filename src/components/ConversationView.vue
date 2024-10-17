<script setup lang="ts">
import { onMounted, ref, watch } from 'vue'
import PerfectScrollbar from 'perfect-scrollbar'
import 'perfect-scrollbar/css/perfect-scrollbar.css'

interface Message {
  id: string;
  role: 'assistant' | 'client';
  content: string;
}

const props = defineProps<{
  messages: Message[]
}>()

const messagesContainer = ref<HTMLElement | null>(null)
let ps: PerfectScrollbar | null = null
const isLoading = ref(false)

onMounted(() => {
  if (messagesContainer.value) {
    ps = new PerfectScrollbar(messagesContainer.value)
  }
})

watch(() => props.messages, (newMessages, oldMessages) => {
  if (newMessages.length > oldMessages.length) {
    isLoading.value = true
    setTimeout(() => {
      isLoading.value = false
      if (ps) {
        ps.update()
        if (messagesContainer.value) {
          messagesContainer.value.scrollTop = messagesContainer.value.scrollHeight
        }
      }
    }, 1000) // Simulating a delay, adjust as needed
  }
}, { deep: true })
</script>

<template>
  <div class="conversation-view">
    <div class="messages" ref="messagesContainer">
      <div 
        v-for="message in messages" 
        :key="message.id"
        :class="['message', message.role]"
      >
        {{ message.content }}
      </div>
      <div v-if="isLoading" class="message loading">
        <div class="loading-icon"></div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.conversation-view {
  flex-grow: 1;
  display: flex;
  flex-direction: column;
  background-color: #fff;
}

.messages {
  flex-grow: 1;
  padding: 20px;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
}

.message {
  max-width: 70%;
  padding: 10px;
  margin-bottom: 10px;
  border-radius: 10px;
}

.message.client {
  background-color: #dcf8c6;
  align-self: flex-end;
  margin-left: auto;
}

.message.assistant {
  background-color: #f0f0f0;
  align-self: flex-start;
}

.message.loading {
  align-self: center;
  background-color: transparent;
  padding: 0;
}

.loading-icon {
  width: 24px;
  height: 24px;
  border: 3px solid #f3f3f3;
  border-top: 3px solid #3498db;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}
</style>