<script>
import ChatMessage from 'widget/components/ChatMessage.vue';
import AgentTypingBubble from 'widget/components/AgentTypingBubble.vue';
import DateSeparator from 'shared/components/DateSeparator.vue';
import Spinner from 'shared/components/Spinner.vue';
import { useDarkMode } from 'widget/composables/useDarkMode';
import { MESSAGE_TYPE } from 'shared/constants/messages';
import { mapActions, mapGetters } from 'vuex';

export default {
  name: 'ConversationWrap',
  components: {
    ChatMessage,
    AgentTypingBubble,
    DateSeparator,
    Spinner,
  },
  props: {
    groupedMessages: {
      type: Array,
      default: () => [],
    },
  },
  setup() {
    const { darkMode } = useDarkMode();
    return { darkMode };
  },
  data() {
    return {
      previousScrollHeight: 0,
      previousConversationSize: 0,
      isNearBottom: true,
      resizeObserver: null,
    };
  },
  computed: {
    ...mapGetters({
      earliestMessage: 'conversation/getEarliestMessage',
      lastMessage: 'conversation/getLastMessage',
      allMessagesLoaded: 'conversation/getAllMessagesLoaded',
      isFetchingList: 'conversation/getIsFetchingList',
      conversationSize: 'conversation/getConversationSize',
      isAgentTyping: 'conversation/getIsAgentTyping',
      conversationAttributes: 'conversationAttributes/getConversationParams',
    }),
    colorSchemeClass() {
      return `${this.darkMode === 'dark' ? 'dark-scheme' : 'light-scheme'}`;
    },
    showStatusIndicator() {
      const { status } = this.conversationAttributes;
      const isConversationInPendingStatus = status === 'pending';
      const isLastMessageIncoming =
        this.lastMessage.message_type === MESSAGE_TYPE.INCOMING;
      return (
        this.isAgentTyping ||
        (isConversationInPendingStatus && isLastMessageIncoming)
      );
    },
  },
  watch: {
    allMessagesLoaded() {
      this.previousScrollHeight = 0;
    },
  },
  mounted() {
    this.$el.addEventListener('scroll', this.handleScroll);
    this.scrollToBottom();
    this.resizeObserver = new ResizeObserver(() => {
      if (this.isNearBottom) this.scrollToBottom();
    });
    this.resizeObserver.observe(this.$refs.conversationWrap);
  },
  updated() {
    if (this.previousConversationSize !== this.conversationSize) {
      this.previousConversationSize = this.conversationSize;
      this.scrollToBottom();
    }
  },
  unmounted() {
    this.$el.removeEventListener('scroll', this.handleScroll);
    this.resizeObserver?.disconnect();
  },
  methods: {
    ...mapActions('conversation', ['fetchOldConversations']),
    scrollToBottom() {
      const container = this.$el;
      container.scrollTop = container.scrollHeight - this.previousScrollHeight;
      this.previousScrollHeight = 0;
    },
    handleScroll() {
      const { scrollTop, scrollHeight, clientHeight } = this.$el;
      this.isNearBottom = scrollHeight - scrollTop - clientHeight < 100;

      if (
        this.isFetchingList ||
        this.allMessagesLoaded ||
        !this.conversationSize
      ) {
        return;
      }

      if (scrollTop < 100) {
        this.fetchOldConversations({ before: this.earliestMessage.id });
        this.previousScrollHeight = this.$el.scrollHeight;
      }
    },
  },
};
</script>

<template>
  <div class="conversation--container" :class="colorSchemeClass">
    <div
      ref="conversationWrap"
      class="conversation-wrap"
      :class="{ 'is-typing': isAgentTyping }"
    >
      <div v-if="isFetchingList" class="message--loader">
        <Spinner />
      </div>
      <div
        v-for="groupedMessage in groupedMessages"
        :key="groupedMessage.date"
        class="messages-wrap"
      >
        <DateSeparator :date="groupedMessage.date" />
        <ChatMessage
          v-for="message in groupedMessage.messages"
          :key="message.id"
          :message="message"
        />
      </div>
      <AgentTypingBubble v-if="showStatusIndicator" />
    </div>
  </div>
</template>

<style scoped lang="scss">
.conversation--container {
  display: flex;
  flex-direction: column;
  flex: 1;
  overflow-y: auto;
  color-scheme: light dark;

  &.light-scheme {
    color-scheme: light;
  }

  &.dark-scheme {
    color-scheme: dark;
  }
}

.conversation-wrap {
  flex: 1;
  @apply px-2 pt-8 pb-2;
}

.message--loader {
  text-align: center;
}
</style>
