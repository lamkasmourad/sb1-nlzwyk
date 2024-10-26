<script setup lang="ts">
import { onMounted, ref, watch, computed } from "vue";
import PerfectScrollbar from "perfect-scrollbar";
import "perfect-scrollbar/css/perfect-scrollbar.css";

interface Conversation {
  id: string;
  title: string;
  date: string;
}

const props = defineProps<{
  conversations: Conversation[];
  selectedConversationId: string | null;
  isLoading: boolean;
  error: string | null;
}>();

const emit = defineEmits(["conversation-client-info"]);

const conversationListRef = ref<HTMLElement | null>(null);
let ps: PerfectScrollbar | null = null;

const groupedConversations = computed(() => {
  const now = new Date();
  const today = new Date(now.getFullYear(), now.getMonth(), now.getDate());
  const yesterday = new Date(today);
  yesterday.setDate(yesterday.getDate() - 1);
  const sevenDaysAgo = new Date(today);
  sevenDaysAgo.setDate(sevenDaysAgo.getDate() - 7);

  const groups: Record<string, Conversation[]> = {
    "Aujourd'hui": [],
    Hier: [],
    "7 derniers jours": [],
  };

  props.conversations.sort(
    (a, b) => new Date(b.date).getTime() - new Date(a.date).getTime()
  );

  for (const conv of props.conversations.slice(0, 30)) {
    const convDate = new Date(conv.date);
    if (convDate >= today) {
      groups["Aujourd'hui"].push(conv);
    } else if (convDate >= yesterday) {
      groups["Hier"].push(conv);
    } else if (convDate >= sevenDaysAgo) {
      groups["7 derniers jours"].push(conv);
    }
  }

  return groups;
});

const formatDate = (dateString: string) => {
  const date = new Date(dateString);
  const now = new Date();
  const today = new Date(now.getFullYear(), now.getMonth(), now.getDate());
  const yesterday = new Date(today);
  yesterday.setDate(yesterday.getDate() - 1);

  if (date >= today) {
    return date.toLocaleTimeString([], { hour: "2-digit", minute: "2-digit" });
  } else if (date >= yesterday) {
    return date.toLocaleTimeString([], { hour: "2-digit", minute: "2-digit" });
  } else {
    return date.toLocaleDateString();
  }
};

onMounted(() => {
  if (conversationListRef.value) {
    ps = new PerfectScrollbar(conversationListRef.value, {
      suppressScrollX: true,
      maxScrollbarLength: 200,
    });
  }
});

watch(
  () => groupedConversations.value,
  () => {
    if (ps) {
      setTimeout(() => {
        ps?.update();
      }, 0);
    }
  },
  { deep: true }
);

watch(
  () => props.conversations,
  () => {
    if (ps) {
      setTimeout(() => {
        ps?.update();
      }, 0);
    }
  },
  { deep: true }
);
</script>

<template>
  <div class="conversation-history">
    <h2>Historique des conversations</h2>
    <div class="conversation-list" ref="conversationListRef">
      <div v-if="isLoading" class="loading-container">
        <div class="loading-spinner"></div>
      </div>
      <div v-else-if="error" class="error-message">
        {{ error }}
      </div>
      <template
        v-else-if="
          Object.values(groupedConversations).some((group) => group.length > 0)
        "
      >
        <template
          v-for="(conversations, label) in groupedConversations"
          :key="label"
        >
          <div v-if="conversations.length" class="group-label">{{ label }}</div>
          <div
            v-for="conversation in conversations"
            :key="conversation.id"
            :class="[
              'conversation-item',
              { selected: conversation.id === selectedConversationId },
            ]"
            @click="emit('conversation-client-info', conversation.id)"
          >
            <div class="conversation-title">{{ conversation.title }}</div>
            <div class="conversation-date">
              {{ formatDate(conversation.date) }}
            </div>
          </div>
        </template>
      </template>
      <div v-else class="no-history-message">Aucun historique trouvé</div>
    </div>
  </div>
</template>

<style scoped>
.conversation-history {
  width: 300px;
  background-color: #fff;
  border-right: 1px solid #e0e0e0;
  display: flex;
  flex-direction: column;
  height: 100vh;
}
.loading-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100%;
}

.loading-spinner {
  width: 40px;
  height: 40px;
  border: 3px solid #f3f3f3;
  border-top: 3px solid #3498db;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}

h2 {
  margin: 0;
  padding: 20px;
  font-size: 18px;
  color: #333;
  border-bottom: 1px solid #e0e0e0;
}

.conversation-list {
  flex-grow: 1;
  overflow-y: auto;
  position: relative;
}

.group-label {
  padding: 10px 20px;
  font-weight: bold;
  background-color: #f0f2f5;
  border-bottom: 1px solid #e0e0e0;
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

.no-history-message {
  padding: 20px;
  text-align: center;
  color: #888;
}
</style>