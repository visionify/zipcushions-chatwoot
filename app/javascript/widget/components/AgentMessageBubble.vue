<script>
import { useMessageFormatter } from 'shared/composables/useMessageFormatter';
import ChatCard from 'shared/components/ChatCard.vue';
import ChatForm from 'shared/components/ChatForm.vue';
import ChatOptions from 'shared/components/ChatOptions.vue';
import ChatArticle from './template/Article.vue';
import EmailInput from './template/EmailInput.vue';
import CustomerSatisfaction from 'shared/components/CustomerSatisfaction.vue';
import IntegrationCard from './template/IntegrationCard.vue';

export default {
  name: 'AgentMessageBubble',
  components: {
    ChatArticle,
    ChatCard,
    ChatForm,
    ChatOptions,
    EmailInput,
    CustomerSatisfaction,
    IntegrationCard,
  },
  props: {
    message: { type: String, default: null },
    contentType: { type: String, default: null },
    messageType: { type: Number, default: null },
    messageId: { type: Number, default: null },
    messageContentAttributes: {
      type: Object,
      default: () => {},
    },
  },
  data() {
    return {
      activeCardIndex: 0,
      showSwipeHint: true,
    };
  },
  setup() {
    const { formatMessage, getPlainText, truncateMessage, highlightContent } =
      useMessageFormatter();
    return {
      formatMessage,
      getPlainText,
      truncateMessage,
      highlightContent,
    };
  },
  computed: {
    isTemplate() {
      return this.messageType === 3;
    },
    isTemplateEmail() {
      return this.contentType === 'input_email';
    },
    isCards() {
      return this.contentType === 'cards';
    },
    isOptions() {
      return this.contentType === 'input_select';
    },
    isForm() {
      return this.contentType === 'form';
    },
    isArticle() {
      return this.contentType === 'article';
    },
    isCSAT() {
      return this.contentType === 'input_csat';
    },
    isIntegrations() {
      return this.contentType === 'integrations';
    },
    cardItems() {
      return this.messageContentAttributes?.items || [];
    },
    hasMultipleCards() {
      return this.cardItems.length > 2;
    },
  },
  mounted() {
    if (this.isCards && this.hasMultipleCards) {
      setTimeout(() => {
        this.showSwipeHint = false;
      }, 5000);
    }
  },
  methods: {
    onResponse(messageResponse) {
      this.$store.dispatch('message/update', messageResponse);
    },
    onOptionSelect(selectedOption) {
      this.onResponse({
        submittedValues: [selectedOption],
        messageId: this.messageId,
      });
    },
    onFormSubmit(formValues) {
      const formValuesAsArray = Object.keys(formValues).map(key => ({
        name: key,
        value: formValues[key],
      }));
      this.onResponse({
        submittedValues: formValuesAsArray,
        messageId: this.messageId,
      });
    },
    scrollCarousel(direction) {
      const track = this.$refs.cardsTrack;
      if (track) {
        track.scrollBy({ left: direction * 185, behavior: 'smooth' });
      }
    },
    onCarouselScroll() {
      const track = this.$refs.cardsTrack;
      if (track) {
        this.activeCardIndex = Math.round(track.scrollLeft / 185);
      }
    },
  },
};
</script>

<template>
  <div class="chat-bubble-wrap">
    <div
      v-if="
        !isCards && !isOptions && !isForm && !isArticle && !isCards && !isCSAT
      "
      class="chat-bubble agent bg-n-background dark:bg-n-solid-3 text-n-slate-12"
    >
      <div
        v-dompurify-html="formatMessage(message, false)"
        class="message-content text-n-slate-12"
      />
      <EmailInput
        v-if="isTemplateEmail"
        :message-id="messageId"
        :message-content-attributes="messageContentAttributes"
      />

      <IntegrationCard
        v-if="isIntegrations"
        :message-id="messageId"
        :meeting-data="messageContentAttributes.data"
      />
    </div>
    <div v-if="isOptions">
      <ChatOptions
        :title="message"
        :options="messageContentAttributes.items"
        :hide-fields="!!messageContentAttributes.submitted_values"
        @option-select="onOptionSelect"
      />
    </div>
    <ChatForm
      v-if="isForm && !messageContentAttributes.submitted_values"
      :items="messageContentAttributes.items"
      :button-label="messageContentAttributes.button_label"
      :submitted-values="messageContentAttributes.submitted_values"
      @submit="onFormSubmit"
    />
    <div v-if="isCards" class="carousel-wrapper">
      <!-- Arrow + Cards Row -->
      <div class="carousel-row">
        <button
          v-if="hasMultipleCards"
          class="carousel-arrow carousel-arrow-left"
          @click="scrollCarousel(-1)"
        >
          ‹
        </button>
        <div
          ref="cardsTrack"
          class="cards-track"
          @scroll="onCarouselScroll"
        >
          <ChatCard
            v-for="item in cardItems"
            :key="item.title"
            :media-url="item.media_url"
            :title="item.title"
            :description="item.description"
            :actions="item.actions"
          />
        </div>
        <button
          v-if="hasMultipleCards"
          class="carousel-arrow carousel-arrow-right"
          @click="scrollCarousel(1)"
        >
          ›
        </button>
      </div>

      <!-- Dot indicators -->
      <div v-if="hasMultipleCards" class="carousel-dots">
        <span
          v-for="(item, index) in cardItems"
          :key="'dot-' + index"
          class="carousel-dot"
          :class="{ active: index === activeCardIndex }"
        />
      </div>

      <!-- Hint -->
      <div v-if="hasMultipleCards && showSwipeHint" class="swipe-hint">
        <span class="swipe-hint-text">← more options →</span>
      </div>
    </div>
    <div v-if="isArticle">
      <ChatArticle :items="messageContentAttributes.items" />
    </div>
    <CustomerSatisfaction
      v-if="isCSAT"
      :message-content-attributes="messageContentAttributes.submitted_values"
      :display-type="messageContentAttributes.display_type"
      :message="message"
      :message-id="messageId"
    />
  </div>
</template>

<style scoped>
/* Carousel wrapper */
.carousel-wrapper {
  max-width: 100%;
  padding: 4px 0;
}

/* Arrow + cards row */
.carousel-row {
  display: flex;
  align-items: center;
  gap: 4px;
}

/* Arrow buttons */
.carousel-arrow {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  border: 1px solid #e0e0e0;
  background: #fff;
  font-size: 18px;
  cursor: pointer;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.08);
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #555;
  transition: all 0.2s;
  user-select: none;
  padding: 0;
  line-height: 1;
}
.carousel-arrow:hover {
  background: #f5f5f5;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.12);
  color: #1b8ceb;
}
.carousel-arrow:active {
  transform: scale(0.95);
}

/* Cards track */
.cards-track {
  display: flex;
  overflow-x: hidden;
  gap: 10px;
  padding: 4px 2px;
  scroll-snap-type: x mandatory;
  -webkit-overflow-scrolling: touch;
  scrollbar-width: none;
  flex: 1;
  scroll-behavior: smooth;
}
.cards-track::-webkit-scrollbar {
  display: none;
}

/* Dot indicators */
.carousel-dots {
  display: flex;
  justify-content: center;
  gap: 5px;
  padding: 8px 0 4px 0;
}
.carousel-dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: #ddd;
  transition: all 0.3s;
}
.carousel-dot.active {
  background: #1b8ceb;
  width: 16px;
  border-radius: 3px;
}

/* Hint */
.swipe-hint {
  text-align: center;
  padding: 4px 0 2px 0;
  animation: fadeHint 5s ease forwards;
}
.swipe-hint-text {
  font-size: 11px;
  color: #aaa;
  letter-spacing: 0.5px;
  animation: bounceHint 1.5s ease-in-out 3;
  display: inline-block;
}
@keyframes bounceHint {
  0%,
  100% {
    transform: translateX(0);
  }
  25% {
    transform: translateX(-5px);
  }
  75% {
    transform: translateX(5px);
  }
}
@keyframes fadeHint {
  0% {
    opacity: 1;
  }
  75% {
    opacity: 1;
  }
  100% {
    opacity: 0;
  }
}

/* Dark mode arrow support */
:global(.dark) .carousel-arrow {
  background: #2a2a2a;
  border-color: #444;
  color: #ccc;
}
:global(.dark) .carousel-arrow:hover {
  background: #333;
  color: #1b8ceb;
}
:global(.dark) .carousel-dot {
  background: #444;
}
</style>
