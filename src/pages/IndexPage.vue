<template>
  <q-page class="flex flex-center">
    <!-- <img
      alt="Quasar logo"
      src="~assets/quasar-logo-vertical.svg"
      style="width: 200px; height: 200px"
    /> -->
    <div style="width: 100%; max-width: 700px" class="self-end q-ma-lg">
      <q-scroll-area
        ref="scrollAreaRef"
        style="height: calc(100vh - 200px)"
        class="q-px-md q-mb-lg"
      >
        <div class="flex flex-center">
          <img
            alt="Quasar logo"
            src="~assets/quasar-logo-vertical.svg"
            style="width: 200px; height: 200px"
            class="full-width q-ma-xl"
          />
          <div v-if="thread.length === 0" class="flex flex-center row">
            <q-chip
              v-for="question in defaultQuestions"
              :key="question"
              class="q-ma-md cursor-pointer"
              @click="
                text = question;
                onSend();
              "
              clickable
            >
              {{ question }}
            </q-chip>
          </div>
        </div>

        <!-- <q-chat-message name="Me" :text="['hey, how are you?']" sent />
        <q-chat-message
          name="Assistant"
          :text="['doing fine, how r you?']"
          text-color="white"
          bg-color="primary"
        /> -->
        <q-chat-message
          v-for="(message, index) in thread"
          :key="message.id"
          :name="message.role === 'user' ? 'Me' : 'Assistant'"
          :text="[message.content[0].text.value.replace(/\n/g, '<br />')]"
          :sent="message.role === 'user'"
          :text-color="message.role === 'user' ? undefined : 'white'"
          :bg-color="message.role === 'user' ? undefined : 'primary'"
          :stamp="
            message.role === 'user' ? `${getSentCount(index)} of 10` : undefined
          "
          text-html
        />
        <q-chat-message
          v-show="isRunning"
          name="Assistant"
          text-color="white"
          bg-color="primary"
        >
          <q-spinner-dots size="1.5rem" />
        </q-chat-message>
      </q-scroll-area>
      <!-- </div> -->

      <q-input
        bottom-slots
        v-model="text"
        counter
        maxlength="2000"
        :placeholder="
          getSentCount(thread.length) >= 10
            ? 'You have reached the limit of 10 messages'
            : 'Ask me anything...'
        "
        :disable="getSentCount(thread.length) >= 10"
        outlined
        clearable
        autogrow
      >
        <template v-slot:append>
          <q-btn v-if="isSending" round dense flat disable>
            <q-spinner />
          </q-btn>
          <q-btn
            v-else
            round
            dense
            flat
            icon="send"
            :color="text ? 'primary' : ''"
            :disable="!!!text || getSentCount(thread.length) >= 10"
            @click="onSend"
          >
            <q-tooltip>Send</q-tooltip>
          </q-btn>
        </template>

        <template v-slot:before>
          <q-btn v-if="isLoadingChat" round dense flat size="lg" disable>
            <q-spinner />
          </q-btn>
          <q-btn
            v-else
            round
            dense
            flat
            icon="maps_ugc"
            color="primary"
            size="lg"
            @click="onNewChat"
          >
            <q-tooltip>New Chat</q-tooltip>
          </q-btn>
        </template>
      </q-input>
    </div>
  </q-page>
</template>

<script setup>
import { onMounted, ref } from "vue";
import { Cookies } from "quasar";
import { api } from "boot/axios";

const defaultQuestions = [
  "How are you?",
  "What is quasar?",
  "What can I do for you?",
  "What is the meaning of life?",
];

const threadId = ref(Cookies.get("threadId"));
const runId = ref(null);
const isLoadingChat = ref(false);
const isRunning = ref(false);
const isSending = ref(false);
const interval = ref(null);

const scrollAreaRef = ref(null);
const text = ref("");
const thread = ref([]);

const scrollToBottom = () => {
  setTimeout(() => {
    scrollAreaRef.value.setScrollPercentage("vertical", 1, 100);
  }, 110);
};

const createThread = async () => {
  const response = await api.post("/threads");
  console.log(response);
  return response.data.id;
};

const getMessages = async (limit = 10) => {
  const response = await api.get(`/threads/${threadId.value}/messages`, {
    params: {
      limit,
    },
  });
  console.log(response);
  return response.data.data;
};

const checkRunStatus = async () => {
  const response = await api.get(
    `/threads/${threadId.value}/runs/${runId.value}`
  );
  console.log(response);
  if (response.data.status === "completed") {
    isRunning.value = false;
    clearInterval(interval.value);

    const messages = await getMessages(1);
    thread.value.push(messages[0]);
    scrollToBottom();
  }
};

const onSend = async () => {
  isSending.value = true;

  try {
    const response = await api.post(`/threads/${threadId.value}/messages`, {
      message: text.value,
    });
    isSending.value = false;
    console.log(response);

    const { run, message } = response.data;
    text.value = "";
    thread.value.push(message);
    isRunning.value = true;
    scrollToBottom();

    runId.value = run.id;
    interval.value = setInterval(checkRunStatus, 5000);
  } catch (error) {
    console.log(error);
    isSending.value = false;
  }
};

const onNewChat = async () => {
  isLoadingChat.value = true;
  threadId.value = await createThread();
  Cookies.set("threadId", threadId.value);
  thread.value = [];
  isLoadingChat.value = false;
};

const getSentCount = (n) => Math.ceil(n / 2) + 1;

onMounted(async () => {
  console.log(threadId.value);
  isLoadingChat.value = true;
  if (!threadId.value) {
    // Create new thread
    threadId.value = await createThread();
    Cookies.set("threadId", threadId.value);
  }
  // Fetch messages
  const messages = await getMessages();
  thread.value = messages.reverse();
  scrollToBottom();
  isLoadingChat.value = false;
});
</script>
