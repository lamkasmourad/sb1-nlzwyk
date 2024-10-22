<script setup lang="ts">
import { onMounted, ref, watch, nextTick, computed, onUnmounted } from "vue";
import axios from "axios";
import PerfectScrollbar from "perfect-scrollbar";
import "perfect-scrollbar/css/perfect-scrollbar.css";

interface Message {
  id: string;
  role: "assistant" | "client";
  content: string;
}

interface ConversationInfo {
  [key: string]: string | undefined;
}


const props = defineProps<{
  conversationInfo: ConversationInfo;
  selectedConversationId: string | null;
  isLoading: boolean;
  audioSrc: string | undefined;
  newCall: boolean | undefined;
}>();

const emit = defineEmits(['newCall']);

const messages = ref<Message[]>([]);
const messagesContainer = ref<HTMLElement | null>(null);
let ps: PerfectScrollbar | null = null;
const audioElement = ref<HTMLAudioElement | null>(null);
const isPlaying = ref(false);
const currentTime = ref(0);
const duration = ref(0);

const formattedCurrentTime = computed(() => {
  return formatTime(currentTime.value);
});

const formattedDuration = computed(() => {
  return formatTime(duration.value || 0);
});

const filteredConversationInfo = computed(() => {
  const filtered: Record<string, string> = {};
  Object.entries(props.conversationInfo).forEach(([key, value]) => {
    if (
      value !== undefined &&
      value !== "" &&
      key !== "contrat" &&
      key !== "contrat_type"
    ) {
      filtered[key] = value;
    }
  });
  return filtered;
});

const agentNumber = ref("1000");
const agentSecret = ref("EcA+1000");
const apiBaseUrl = import.meta.env.VITE_API_BASE_URL;

const softphoneUrl = computed(() => {
  return `https://softphone-assistant.teletech-int.info/?agentNumber=${encodeURIComponent(agentNumber.value)}&agentSecret=${encodeURIComponent(agentSecret.value)}`;
});

function formatTime(time: number): string {
  if (isNaN(time) || time === 0) {
    return "00:00";
  }
  const minutes = Math.floor(time / 60);
  const seconds = Math.floor(time % 60);
  return `${minutes.toString().padStart(2, "0")}:${seconds
    .toString()
    .padStart(2, "0")}`;
}

let socket: WebSocket | null = null;
const SOCKET_URL = import.meta.env.VITE_ENV === 'local' ? 'ws://172.30.1.25:8077' : 'wss://voice-assistant.teletech-int.info/ws';
const RECONNECT_INTERVAL = 5000;

function connectWebSocket() {
  socket = new WebSocket(SOCKET_URL);

  socket.onopen = () => {
    console.log('WebSocket connected');
  };

  socket.onmessage = (event) => {
    console.log("New message arrived");
    const data = JSON.parse(event.data);
    console.log(data)
    if (data.callId && data.message) {
      console.log("push new message",data.message.content)
      messages.value.push({
        id: data.callId,
        role: data.message.role == 'user' ? 'client' : data.message.role,
        content: data.message.content
      });
      scrollToBottom();
    }
  };

  socket.onclose = (event) => {
    console.log('WebSocket disconnected:', event.reason);
    socket = null;
    setTimeout(connectWebSocket, RECONNECT_INTERVAL);
  };

  socket.onerror = (error) => {
    console.error('WebSocket error:', error);
    socket?.close();
  };
}

onMounted(() => {
  if (messagesContainer.value) {
    ps = new PerfectScrollbar(messagesContainer.value);
    scrollToBottom();
  }
  if (audioElement.value) {
    audioElement.value.addEventListener("loadedmetadata", () => {
      duration.value = audioElement.value?.duration || 0;
    });
  }
  window.addEventListener("message", (event) => {
    let data = event.data;
    if (data.action === 'create') {
      console.log( data.action_detail);
      emit('newCall', data);
    }
  });

  connectWebSocket();
});

onUnmounted(() => {
  socket?.close();
});

const scrollToBottom = () => {
  nextTick(() => {
    if (messagesContainer.value) {
      messagesContainer.value.scrollTop = messagesContainer.value.scrollHeight;
      if (ps) {
        ps.update();
      }
    }
  });
};

watch(
  () => props.selectedConversationId,
  async (newConversationId) => {
    if (newConversationId) {
      try {
        const response = await axios.post(`${apiBaseUrl}/api/simulation/dialogues-by-conversation`, {
          conversation_id: newConversationId,
        });
        messages.value = response.data;
      } catch (error) {
        console.error("Error fetching messages:", error);
        messages.value = [];
      }
    }
  }
);

const togglePlay = () => {
  if (audioElement.value) {
    if (isPlaying.value) {
      audioElement.value.pause();
    } else {
      audioElement.value.play();
    }
    isPlaying.value = !isPlaying.value;
  }
};

const updateProgress = () => {
  if (audioElement.value) {
    currentTime.value = audioElement.value.currentTime;
    duration.value = audioElement.value.duration || 0;
  }
};

const setProgress = (event: MouseEvent) => {
  const progressBar = event.currentTarget as HTMLDivElement;
  const rect = progressBar.getBoundingClientRect();
  const clickPosition = event.clientX - rect.left;
  const clickPercentage = clickPosition / rect.width;
  if (audioElement.value) {
    audioElement.value.currentTime =
      clickPercentage * (audioElement.value.duration || 0);
  }
};
</script>

<template>
  <div class="conversation-view">
    <div v-if="selectedConversationId && !props.newCall" class="header">
      <div class="audio-player">
        <button @click="togglePlay" class="play-button">
          {{ isPlaying ? "❚❚" : " ▶" }}
        </button>
        <div class="progress-bar" @click="setProgress">
          <div
            class="progress"
            :style="{ width: `${(currentTime / duration || 0) * 100}%` }"
          ></div>
        </div>
        <div class="time-display">
          {{ formattedCurrentTime }} / {{ formattedDuration }}
        </div>
        <audio
          ref="audioElement"
          :src="props.audioSrc"
          @timeupdate="updateProgress"
          @ended="isPlaying = false"
        ></audio>
      </div>
      <div class="tags">
        <span
          v-for="(value, key) in filteredConversationInfo"
          :key="key"
          class="tag"
        >
          {{ key.charAt(0).toUpperCase() + key.slice(1) }}: {{ value }}
        </span>
      </div>
    </div>
    <div class="messages" ref="messagesContainer">
      <div v-if="isLoading" class="loading-container">
        <div class="loading-icon"></div>
      </div>
      <template v-else>
        <div
          v-if="selectedConversationId && messages.length === 0 && !props.newCall"
          class="no-messages"
        >
          Aucune conversation entre l'assistant et le client n'a été trouvée
        </div>
        <div
          v-for="message in messages"
          :key="message.id"
          :class="['message', message.role]"
        >
          <template v-if="message.content.trim()">
            {{ message.content }}
          </template>
        </div>
        <div v-if="props.newCall" :class="['message', messages.length % 2 === 0 ? 'assistant' : 'client']">
          <div class="loading-dots">...</div>
        </div>
      </template>
    </div>
    <iframe :src="softphoneUrl" class="softphone-iframe" allow="microphone"></iframe>
  </div>
</template>

<style scoped>
.conversation-view {
  flex-grow: 1;
  display: flex;
  flex-direction: column;
  background-color: #fff;
  height: 100vh;
  overflow: hidden;
}

.header {
  height: 100px;
  padding: 20px;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  background-color: #fff;
  border-bottom: 1px solid #d1d5db;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1),
    0 2px 4px -1px rgba(0, 0, 0, 0.06);
}

.audio-player {
  display: flex;
  align-items: center;
  margin-bottom: 15px;
}

.play-button {
  background-color: #3498db;
  color: white;
  border: none;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  font-size: 18px;
  cursor: pointer;
  margin-right: 10px;
}

.progress-bar {
  flex-grow: 1;
  height: 6px;
  background-color: #e0e0e0;
  border-radius: 3px;
  cursor: pointer;
  position: relative;
}

.progress {
  height: 100%;
  background-color: #3498db;
  border-radius: 3px;
  transition: width 0.1s linear;
}

.time-display {
  margin-left: 10px;
  font-size: 0.9em;
  color: #4b5563;
}

.tags {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}

.tag {
  padding: 5px 10px;
  border-radius: 15px;
  background-color: #fff;
  border: 1px solid #d1d5db;
  font-size: 0.9em;
  color: #4b5563;
}

.messages {
  flex-grow: 1;
  overflow-y: auto;
  padding: 20px;
  display: flex;
  flex-direction: column;
}

.message {
  max-width: 600px;
  padding: 10px;
  margin-bottom: 10px;
  border-radius: 10px;
  word-wrap: break-word;
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

.loading-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100%;
}

.loading-icon {
  width: 24px;
  height: 24px;
  border: 3px solid #f3f3f3;
  border-top: 3px solid #3498db;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

.no-messages {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100%;
  font-size: 1.2em;
  color: #4b5563;
  text-align: center;
}

.softphone-iframe {
  width: 100%;
  height: 300px;
  border: none;
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}

.loading-dots {
  font-size: 24px;
  animation: blink 1s infinite;
}

@keyframes blink {
  0%, 100% { opacity: 0; }
  50% { opacity: 1; }
}
</style>