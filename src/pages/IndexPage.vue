<template>
  <q-page class="flex flex-center">
    <!-- <img
      alt="Quasar logo"
      src="~assets/quasar-logo-vertical.svg"
      style="width: 200px; height: 200px"
    /> -->
    <div style="width: 100%; max-width: 700px" class="self-end q-ma-lg">
      <!-- <div class="q-px-md q-py-lg"> -->
      <q-scroll-area
        ref="scrollAreaRef"
        style="height: calc(100vh - 200px)"
        class="q-px-md q-mb-lg"
        visible
      >
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
          <q-btn
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
          <q-btn round dense flat icon="maps_ugc" color="primary" size="lg">
            <q-tooltip>New Chat</q-tooltip>
          </q-btn>
        </template>
      </q-input>
    </div>
  </q-page>
</template>

<script setup>
import { onMounted, ref, nextTick, computed } from "vue";
import { Cookies } from "quasar";
import axios from "axios";

const threadId = ref(Cookies.get("threadId"));
const runId = ref(null);
const isRunning = ref(false);
const interval = ref(null);

const scrollAreaRef = ref(null);
const text = ref("");
const thread = ref([]);

const scrollToBottom = () => {
  setTimeout(() => {
    scrollAreaRef.value.setScrollPercentage("vertical", 1, 100);
  }, 1);
};

const createThread = async () => {
  const response = await axios.post("http://127.0.0.1:8000/threads");
  console.log(response);
  return response.data.id;
};

const getMessages = async () => {
  const response = await axios.get(
    `http://127.0.0.1:8000/threads/${threadId.value}/messages`
  );
  console.log(response);
  return response.data.data;
};

const checkRunStatus = async () => {
  const response = await axios.get(
    `http://127.0.0.1:8000/threads/${threadId.value}/runs/${runId.value}`
  );
  console.log(response);
  if (response.data.status === "completed") {
    isRunning.value = false;
    clearInterval(interval.value);

    const messages = await getMessages();
    thread.value.push(messages[0]);
    // thread.value = messages.reverse();
    scrollToBottom();
  }
};

const onSend = async () => {
  const message = text.value;
  text.value = "";
  thread.value.push({
    role: "user",
    content: [
      {
        type: "text",
        text: {
          value: message,
        },
      },
    ],
  });
  isRunning.value = true;
  scrollToBottom();
  const response = await axios.post(
    `http://127.0.0.1:8000/threads/${threadId.value}/messages`,
    {
      message,
    }
  );
  console.log(response);
  runId.value = response.data.id;

  interval.value = setInterval(checkRunStatus, 5000);
};

const getSentCount = (n) => Math.ceil(n / 2) + 1;

onMounted(async () => {
  console.log(threadId.value);
  if (!threadId.value) {
    // Create new thread
    threadId.value = await createThread();
    Cookies.set("threadId", threadId.value);
  }
  // Fetch messages
  const messages = await getMessages();
  thread.value = messages.reverse();
  scrollToBottom();
  // await nextTick(scrollToBottom);
});
</script>
