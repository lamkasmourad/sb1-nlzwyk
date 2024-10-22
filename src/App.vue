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

const conversations = ref<Conversation[]>([]);
const currentConversationId = ref<string | null>(null);
const isDialoguesLoading = ref(false);
const conversationTags = ref({});
const currentAudioSrc = ref<string | undefined>(undefined);
const isNewCall = ref(false);
const isHistoryLoading = ref(false);
const historyError = ref<string | null>(null);

const currentConversationInfo = computed(() => {
  return conversationTags.value;
});

const apiBaseUrl = import.meta.env.VITE_API_BASE_URL;

onMounted(async () => {
  isHistoryLoading.value = true;
  historyError.value = null;
  try {
    const response = await axios.post(
      `${apiBaseUrl}/api/simulation/recent-conversations`
    );
    conversations.value = response.data;
  } catch (error) {
    console.error("Error fetching conversations:", error);
    historyError.value = "Une erreur s'est produite lors du chargement de l'historique des conversations.";
  } finally {
    isHistoryLoading.value = false;
  }
});

const selectConversation = async (conversationId: string) => {
  currentConversationId.value = conversationId;
  isDialoguesLoading.value = true;
  isNewCall.value = false;
  try {
    const tagsResponse = await axios.post(`${apiBaseUrl}/api/simulation/conversation-client-info`, {
      conversation_id: conversationId,
    });
    conversationTags.value = tagsResponse.data;
    const selectedConversation = conversations.value.find(
      (conv) => conv.id === conversationId
    );
    currentAudioSrc.value = selectedConversation?.audio_path;
  } catch (error) {
    console.error("Error fetching data:", error);
    conversationTags.value = {};
    currentAudioSrc.value = undefined;
  } finally {
    isDialoguesLoading.value = false;
  }
};

const handleNewCall = () => {
  const newConversation: Conversation = {
    id: Date.now().toString(),
    title: "Nouvel appel",
    date: new Date(Date.now() + 3600000).toISOString(),
  };
  conversations.value.unshift(newConversation);
  currentConversationId.value = newConversation.id;
  conversationTags.value = {};
  currentAudioSrc.value = undefined;
  isNewCall.value = true;
};
</script>

<template>
  <div class="chat-app">
    <ConversationHistory
      :conversations="conversations"
      :selectedConversationId="currentConversationId"
      :isLoading="isHistoryLoading"
      :error="historyError"
      @conversation-client-info="selectConversation"
    />
    <ConversationView
      :selectedConversationId="currentConversationId"
      :conversationInfo="currentConversationInfo"
      :isLoading="isDialoguesLoading"
      :audioSrc="currentAudioSrc"
      :newCall="isNewCall"
      @newCall="handleNewCall"
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