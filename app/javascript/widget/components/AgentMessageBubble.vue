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
      touchStartX: null,
      touchStartY: null,
      isSwiping: false,
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
      return this.cardItems.length > 1;
    },
    totalCards() {
      return this.cardItems.length;
    },
    maxIndex() {
      // Last position: show the last 2 cards
      return Math.max(0, this.totalCards - 2);
    },
    trackStyle() {
      // Each card = 50% of viewport. Slide by 50% per step.
      var offset = this.activeCardIndex * 50;
      return {
        transform: 'translateX(-' + offset + '%)',
        transition: 'transform 0.35s cubic-bezier(0.25, 0.46, 0.45, 0.94)',
      };
    },
    canGoPrev() {
      return this.activeCardIndex > 0;
    },
    canGoNext() {
      return this.activeCardIndex < this.maxIndex;
    },
  },
  mounted() {
    if (this.isCards && this.hasMultipleCards) {
      setTimeout(() => {
        this.showSwipeHint = false;
      }, 5000);

      // CRITICAL: Add non-passive touch listeners so preventDefault() actually works
      // Vue's @touchmove.prevent does NOT work in modern browsers (passive by default)
      this.$nextTick(() => {
        var viewport = this.$refs.cardsViewport;
        if (viewport) {
          viewport.addEventListener('touchstart', this.handleTouchStart, { passive: false });
          viewport.addEventListener('touchmove', this.handleTouchMove, { passive: false });
          viewport.addEventListener('touchend', this.handleTouchEnd, { passive: false });
        }
      });
    }
  },
  beforeUnmount() {
    var viewport = this.$refs.cardsViewport;
    if (viewport) {
      viewport.removeEventListener('touchstart', this.handleTouchStart);
      viewport.removeEventListener('touchmove', this.handleTouchMove);
      viewport.removeEventListener('touchend', this.handleTouchEnd);
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
    goToCard(index) {
      if (index >= 0 && index <= this.maxIndex) {
        this.activeCardIndex = index;
      }
    },
    prevCard() {
      if (this.canGoPrev) {
        this.activeCardIndex--;
      }
    },
    nextCard() {
      if (this.canGoNext) {
        this.activeCardIndex++;
      }
    },
    handleTouchStart(e) {
      this.touchStartX = e.touches[0].clientX;
      this.touchStartY = e.touches[0].clientY;
      this.isSwiping = false;
    },
    handleTouchMove(e) {
      if (this.touchStartX === null) return;

      var diffX = Math.abs(e.touches[0].clientX - this.touchStartX);
      var diffY = Math.abs(e.touches[0].clientY - this.touchStartY);

      // If horizontal movement > vertical, it's a card swipe — block scroll
      if (diffX > diffY && diffX > 10) {
        this.isSwiping = true;
        e.preventDefault();
        e.stopPropagation();
      }
    },
    handleTouchEnd(e) {
      if (this.touchStartX === null) return;

      var diff = this.touchStartX - e.changedTouches[0].clientX;

      if (this.isSwiping) {
        if (diff > 40) {
          this.nextCard();
        } else if (diff < -40) {
          this.prevCard();
        }
        e.preventDefault();
        e.stopPropagation();
      }

      this.touchStartX = null;
      this.touchStartY = null;
      this.isSwiping = false;
    },
    onCardSelect(payload) {
      this.onResponse({
        submittedValues: [{ title: payload, value: payload }],
        messageId: this.messageId,
      });
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
      <div class="carousel-row">
        <button
          v-if="hasMultipleCards"
          class="carousel-arrow carousel-arrow-left"
          :class="{ disabled: !canGoPrev }"
          @click="prevCard"
        >
          ‹
        </button>
        <div
          ref="cardsViewport"
          class="cards-viewport"
        >
          <div class="cards-track" :style="trackStyle">
            <div
              v-for="item in cardItems"
              :key="item.title"
              class="card-slide"
            >
              <ChatCard
                :media-url="item.media_url"
                :title="item.title"
                :description="item.description"
                :actions="item.actions"
                @select="onCardSelect"
              />
            </div>
          </div>
        </div>
        <button
          v-if="hasMultipleCards"
          class="carousel-arrow carousel-arrow-right"
          :class="{ disabled: !canGoNext }"
          @click="nextCard"
        >
          ›
        </button>
      </div>

      <div v-if="hasMultipleCards" class="carousel-dots">
        <span
          v-for="(item, index) in cardItems"
          :key="'dot-' + index"
          class="carousel-dot"
          :class="{ active: index === activeCardIndex }"
          @click="goToCard(index)"
        />
      </div>

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
.carousel-wrapper {
  max-width: 100%;
  padding: 4px 0;
}

.carousel-row {
  display: flex;
  align-items: center;
  gap: 2px;
}

.carousel-arrow {
  width: 26px;
  height: 26px;
  border-radius: 50%;
  border: 1px solid #e0e0e0;
  background: #fff;
  font-size: 16px;
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
  z-index: 2;
}
.carousel-arrow:hover {
  background: #f5f5f5;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.12);
  color: #1b8ceb;
}
.carousel-arrow:active {
  transform: scale(0.95);
}
.carousel-arrow.disabled {
  opacity: 0.3;
  cursor: default;
  pointer-events: none;
}

/* Viewport — clips cards, NO scroll, NO touch propagation */
.cards-viewport {
  overflow: hidden;
  flex: 1;
  touch-action: none;
}

/* Track — moves via translateX */
.cards-track {
  display: flex;
  will-change: transform;
}

/* Each card = 50% width so 2 cards show at once */
.card-slide {
  min-width: 50%;
  max-width: 50%;
  flex-shrink: 0;
  padding: 4px 4px;
  box-sizing: border-box;
}

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
  cursor: pointer;
}
.carousel-dot.active {
  background: #1b8ceb;
  width: 16px;
  border-radius: 3px;
}

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
