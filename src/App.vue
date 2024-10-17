<script setup lang="ts">
import ConversationHistory from './components/ConversationHistory.vue'
import ConversationView from './components/ConversationView.vue'
import { ref, onMounted, watchEffect } from 'vue'
import { io } from 'socket.io-client'

interface Conversation {
  id: string;
  title: string;
  date: string; // Add this line
}

interface Message {
  id: string;
  role: 'assistant' | 'client';
  content: string;
}

const conversations = ref<Conversation[]>([])
const currentConversation = ref<Message[]>([])
const currentConversationId = ref<string | null>(null)
const socket = io('YOUR_WEBSOCKET_SERVER_URL')

onMounted(() => {
  // Fetch conversation history
  // This is a placeholder. In a real application, you would fetch this from your backend
  conversations.value = [
    { id: '1', title: 'First Conversation', date: '2023-06-01' },
    { id: '2', title: 'Second Conversation', date: '2023-06-02' },
  ]

  // Listen for new messages from the websocket
  socket.on('new_message', (message: Message) => {
    if (message.id === currentConversationId.value) {
      currentConversation.value.push(message)
    }
  })
})

const selectConversation = (conversationId: string) => {
  currentConversationId.value = conversationId
  // Fetch conversation messages
  // This is a placeholder. In a real application, you would fetch this from your backend
  if (conversationId === '1') {
    currentConversation.value = [
      { id: '1', role: 'client', content: 'Hello, how can I help you?' },
      { id: '2', role: 'assistant', content: 'Hi there! How can I assist you today?' },
    ]
  } else if (conversationId === '2') {
    currentConversation.value = [
      { id: '1', role: 'client', content: 'I have a question about Vue.js' },
      { id: '2', role: 'assistant', content: "Sure, I'd be happy to help with Vue.js. What would you like to know?" },
    ]
  }
}

// Simulate receiving messages (remove this in production)
watchEffect(() => {
  if (currentConversationId.value) {
    const interval = setInterval(() => {
      const newMessage: Message = {
        id: Date.now().toString(),
        role: Math.random() > 0.5 ? 'client' : 'assistant',
        content: `This is a simulated ${Math.random() > 0.5 ? 'voice' : 'text'} message at ${new Date().toLocaleTimeString()}`
      }
      currentConversation.value.push(newMessage)
    }, 5000)

    return () => clearInterval(interval)
  }
})
</script>

<template>
  <div class="chat-app">
    <ConversationHistory 
      :conversations="conversations"
      :selectedConversationId="currentConversationId"
      @select-conversation="selectConversation"
    />
    <ConversationView 
      :messages="currentConversation"
    />
  </div>
</template>

<style>
body {
  margin: 0;
  font-family: Arial, sans-serif;
  background-color: #f0f2f5;
}

.chat-app {
  display: flex;
  height: 100vh;
}
</style>