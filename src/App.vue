<script setup lang="ts">
import ConversationHistory from "./components/ConversationHistory.vue";
import ConversationView from "./components/ConversationView.vue";
import { ref, onMounted, computed } from "vue";
import axios from "axios";

interface Conversation {
  id: string;
  title: string;
  date: string;
  audio_path?: string;
  status?: string;
  client?: string;
  ref?: string;
}

interface Message {
  id: string;
  role: "assistant" | "client";
  content: string;
}

const conversations = ref<Conversation[]>([]);
const currentConversation = ref<Message[]>([]);
const currentConversationId = ref<string | null>(null);
const isDialoguesLoading = ref(false);
const conversationTags = ref({});
const currentAudioSrc = ref<string | undefined>(undefined);

const currentConversationInfo = computed(() => {
  return conversationTags.value;
});

onMounted(async () => {
  try {
    const response = await axios.post(
      "http://127.0.0.1:8000/api/simulation/recent-conversations"
    );
    conversations.value = response.data;
  } catch (error) {
    console.error("Error fetching conversations:", error);
  }
});

const selectConversation = async (conversationId: string) => {
  currentConversationId.value = conversationId;
  isDialoguesLoading.value = true;
  try {
    const [dialoguesResponse, tagsResponse] = await Promise.all([
      axios.post(
        "http://127.0.0.1:8000/api/simulation/dialogues-by-conversation",
        { conversation_id: conversationId }
      ),
      axios.post(
        "http://127.0.0.1:8000/api/simulation/conversation-client-info",
        { conversation_id: conversationId }
      ),
    ]);
    currentConversation.value = dialoguesResponse.data;
    conversationTags.value = tagsResponse.data;
    const selectedConversation = conversations.value.find(
      (conv) => conv.id === conversationId
    );
    currentAudioSrc.value = selectedConversation?.audio_path;
  } catch (error) {
    console.error("Error fetching data:", error);
    currentConversation.value = [];
    conversationTags.value = {};
    currentAudioSrc.value = undefined;
  } finally {
    isDialoguesLoading.value = false;
  }
};
</script>

<template>
  <div class="chat-app">
    <ConversationHistory
      :conversations="conversations"
      :selectedConversationId="currentConversationId"
      @conversation-client-info="selectConversation"
    />
    <ConversationView
      :messages="currentConversation"
      :selectedConversationId="currentConversationId"
      :conversationInfo="currentConversationInfo"
      :isLoading="isDialoguesLoading"
      :audioSrc="currentAudioSrc"
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
  overflow: hidden;
}

html {
  overflow: hidden;
}

.chat-app > * {
  flex-shrink: 0;
}

.chat-app > :first-child {
  width: 300px;
}

.chat-app > :last-child {
  flex-grow: 1;
  overflow: auto;
}
</style>