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
        class="q-mb-lg"
      >
        <div class="flex flex-center">
          <img
            alt="Quasar logo"
            src="~assets/quasar-logo-vertical.svg"
            style="width: 200px; height: 200px"
            class="full-width q-ma-xl"
          />
          <div
            v-if="thread.length === 0 && !isLoadingChat"
            class="flex flex-center row"
          >
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

        <transition-group
          appear
          enter-active-class="animated zoomIn"
          leave-active-class="animated zoomOut"
        >
          <q-chat-message
            v-for="(message, index) in thread"
            :key="message.id"
            :name="message.role === 'user' ? 'Me' : 'Assistant'"
            :text="[converter.makeHtml(message.content[0].text.value)]"
            :sent="message.role === 'user'"
            :text-color="message.role === 'user' ? 'white' : undefined"
            :bg-color="message.role === 'user' ? 'primary' : 'grey-4'"
            :stamp="
              message.role === 'user'
                ? `${getSentCount(index + 1)} of 10`
                : undefined
            "
            class="q-px-md"
            text-html
          />
          <q-chat-message
            key="running"
            v-show="isRunning"
            name="Assistant"
            class="q-px-md"
            bg-color="grey-4"
          >
            <q-spinner-dots size="1.5rem" />
          </q-chat-message>
        </transition-group>
        <q-page-sticky position="bottom" :offset="[0, 10]">
          <q-btn
            v-if="
              thread.length > 0 &&
              scrollAreaRef.getScrollTarget().scrollTop + 1000 <
                scrollAreaRef.getScrollTarget().scrollHeight
            "
            round
            dense
            icon="arrow_downward"
            color="primary"
            @click="scrollAndHighlight(false)"
          >
            <q-tooltip>Scroll to bottom</q-tooltip>
          </q-btn>
        </q-page-sticky>
      </q-scroll-area>
      <!-- </div> -->

      <q-input
        bottom-slots
        v-model="text"
        counter
        maxlength="2000"
        :placeholder="
          getSentCount(thread.length) >= 10
            ? 'You have reached the limit of 10 messages. Please start a new chat.'
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

<style lang="scss">
.q-message-text-content--received a {
  color: $primary;
  text-decoration: inherit;
  font-weight: 500;
}

/** Gitgub-style formatting of code in pre tags */
.q-message-text-content--received pre {
  background-color: #fff !important;
  border-radius: 3px;
  font-size: 85%;
  line-height: 1.45;
  overflow: auto;
  padding: 16px;
  margin: 10px;
  word-wrap: normal;
}
</style>

<script setup>
import { onMounted, ref } from "vue";
import { Cookies } from "quasar";
import { api } from "boot/axios";
import { Converter } from "showdown";
import Prism from "prismjs";
import "prismjs/themes/prism.css";

const defaultQuestions = [
  "What is the Quasar Framework?",
  "Where can I find the Quasar Documentation?",
  "How do I install Quasar?",
  "How can I get help?",
];

const converter = new Converter();

const threadId = ref(Cookies.get("threadId"));
const runId = ref(null);
const isLoadingChat = ref(false);
const isRunning = ref(false);
const isSending = ref(false);

const scrollAreaRef = ref(null);
const text = ref("");
const thread = ref([]);

const scrollAndHighlight = (highlight = true) => {
  setTimeout(() => {
    scrollAreaRef.value.setScrollPercentage("vertical", 1, 100);
    if (highlight) {
      Prism.highlightAll();
    }
  }, 110);
};

const createThread = async () => {
  const response = await api.post("/threads");
  console.log(response);
  return response.data.id;
};

const getMessages = async (limit = 20) => {
  const response = await api.get(`/threads/${threadId.value}/messages`, {
    params: {
      limit,
    },
  });
  console.log(response);
  return response.data.data;
};

const pollRunStatus = async () => {
  // Poll using while loop to ensure each call is completed before the next
  while (isRunning.value) {
    const response = await api.get(
      `/threads/${threadId.value}/runs/${runId.value}`
    );
    console.log(response);
    if (response.data.status === "completed") {
      // const messages = await getMessages(1);
      // thread.value.push(messages[0]);
      thread.value.push(...(await getMessages(1)));
      scrollAndHighlight();
      isRunning.value = false;
    }
    await new Promise((resolve) => setTimeout(resolve, 5000));
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
    scrollAndHighlight();

    runId.value = run.id;
    pollRunStatus();
  } catch (error) {
    console.log(error);
    isSending.value = false;
  }
};

const onNewChat = async () => {
  isRunning.value = false;
  isSending.value = false;
  isLoadingChat.value = true;
  threadId.value = await createThread();
  Cookies.set("threadId", threadId.value);
  thread.value = [];
  isLoadingChat.value = false;
};

const getSentCount = (n) => Math.ceil(n / 2);

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
  scrollAndHighlight();
  isLoadingChat.value = false;
});
</script>
