<script setup lang="ts">
import { onMounted, ref, watch } from 'vue'
import PerfectScrollbar from 'perfect-scrollbar'
import 'perfect-scrollbar/css/perfect-scrollbar.css'

interface Conversation {
  id: string;
  title: string;
  date: string; // Add this line
}

const props = defineProps<{
  conversations: Conversation[]
  selectedConversationId: string | null
}>()

const emit = defineEmits(['select-conversation'])

const conversationListRef = ref<HTMLElement | null>(null)
let ps: PerfectScrollbar | null = null

onMounted(() => {
  if (conversationListRef.value) {
    ps = new PerfectScrollbar(conversationListRef.value)
  }
})

watch(() => props.conversations, () => {
  if (ps) {
    setTimeout(() => {
      ps.update()
    }, 0)
  }
}, { deep: true })
</script>

<template>
  <div class="conversation-history">
    <h2>Conversation History</h2>
    <div class="conversation-list" ref="conversationListRef">
      <div 
        v-for="conversation in conversations" 
        :key="conversation.id"
        :class="['conversation-item', { 'selected': conversation.id === selectedConversationId }]"
        @click="emit('select-conversation', conversation.id)"
      >
        <div class="conversation-title">{{ conversation.title }}</div>
        <div class="conversation-date">{{ conversation.date }}</div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.conversation-history {
  width: 250px;
  background-color: #fff;
  border-right: 1px solid #e0e0e0;
}

h2 {
  margin: 0;
  padding: 20px;
  font-size: 18px;
  color: #333;
  border-bottom: 1px solid #e0e0e0;
}

.conversation-list {
  height: calc(100vh - 59px); /* Adjusted to account for the h2 height */
  overflow-y: auto;
}

.conversation-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px 20px;
  cursor: pointer;
  border-bottom: 1px solid #e0e0e0;
  transition: background-color 0.3s ease;
}

.conversation-item:hover {
  background-color: #f0f2f5;
}

.conversation-item.selected {
  background-color: #e6f7ff;
  font-weight: bold;
}

.conversation-title {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  flex-grow: 1;
  margin-right: 10px;
}

.conversation-date {
  font-size: 0.8em;
  color: #888;
  white-space: nowrap;
}
</style>