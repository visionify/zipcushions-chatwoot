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
      currentPage: 0,
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
      return this.cardItems.length > 2;
    },
    totalCards() {
      return this.cardItems.length;
    },
    cardsPerPage() {
      return 2;
    },
    totalPages() {
      return Math.ceil(this.totalCards / this.cardsPerPage);
    },
    trackStyle() {
      var offset = this.currentPage * 100;
      return {
        transform: 'translateX(-' + offset + '%)',
        transition: 'transform 0.35s cubic-bezier(0.25, 0.46, 0.45, 0.94)',
      };
    },
    canGoPrev() {
      return this.currentPage > 0;
    },
    canGoNext() {
      return this.currentPage < this.totalPages - 1;
    },
  },
  mounted() {
    if (this.isCards && this.totalCards > 2) {
      setTimeout(() => {
        this.showSwipeHint = false;
      }, 5000);

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
    goToPage(pageIndex) {
      if (pageIndex >= 0 && pageIndex < this.totalPages) {
        this.currentPage = pageIndex;
      }
    },
    prevPage() {
      if (this.canGoPrev) {
        this.currentPage--;
      }
    },
    nextPage() {
      if (this.canGoNext) {
        this.currentPage++;
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
          this.nextPage();
        } else if (diff < -40) {
          this.prevPage();
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
      <!-- Arrow buttons OUTSIDE the viewport, positioned absolutely -->
      <div class="carousel-container">
        <button
          v-if="hasMultipleCards"
          class="carousel-arrow carousel-arrow-left"
          :class="{ disabled: !canGoPrev }"
          @click="prevPage"
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
          @click="nextPage"
        >
          ›
        </button>
      </div>

      <!-- Page dots (not per-card dots) -->
      <div v-if="hasMultipleCards" class="carousel-dots">
        <span
          v-for="page in totalPages"
          :key="'page-' + page"
          class="carousel-dot"
          :class="{ active: (page - 1) === currentPage }"
          @click="goToPage(page - 1)"
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
  overflow: hidden;
}

.carousel-container {
  position: relative;
  display: flex;
  align-items: center;
}

.carousel-arrow {
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  width: 24px;
  height: 24px;
  border-radius: 50%;
  border: 1px solid #e0e0e0;
  background: rgba(255, 255, 255, 0.92);
  font-size: 15px;
  cursor: pointer;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #555;
  transition: all 0.2s;
  user-select: none;
  padding: 0;
  line-height: 1;
  z-index: 5;
}
.carousel-arrow-left {
  left: 2px;
}
.carousel-arrow-right {
  right: 2px;
}
.carousel-arrow:hover {
  background: #f5f5f5;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.12);
  color: #1b8ceb;
}
.carousel-arrow:active {
  transform: translateY(-50%) scale(0.95);
}
.carousel-arrow.disabled {
  opacity: 0.3;
  cursor: default;
  pointer-events: none;
}

/* Viewport — takes full width, clips cards, NO scroll */
.cards-viewport {
  overflow: hidden;
  width: 100%;
  touch-action: none;
}

/* Track — holds all cards in a row, slides by page (100%) */
.cards-track {
  display: flex;
  will-change: transform;
}

/* Each card = exactly 50% so 2 cards fill the viewport */
.card-slide {
  min-width: 50%;
  max-width: 50%;
  flex-shrink: 0;
  padding: 4px;
  box-sizing: border-box;
}

/* Force card images and content to stay within bounds */
.card-slide :deep(.chat-card) {
  max-width: 100%;
  overflow: hidden;
}
.card-slide :deep(.chat-card img) {
  width: 100%;
  height: auto;
  object-fit: cover;
  max-height: 120px;
}
.card-slide :deep(.chat-card .title) {
  font-size: 13px;
  line-height: 1.3;
  overflow: hidden;
  text-overflow: ellipsis;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
}
.card-slide :deep(.chat-card .description) {
  font-size: 11px;
  line-height: 1.3;
  overflow: hidden;
  text-overflow: ellipsis;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
}

/* Page dots */
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
  0%, 100% { transform: translateX(0); }
  25% { transform: translateX(-5px); }
  75% { transform: translateX(5px); }
}
@keyframes fadeHint {
  0% { opacity: 1; }
  75% { opacity: 1; }
  100% { opacity: 0; }
}

:global(.dark) .carousel-arrow {
  background: rgba(42, 42, 42, 0.92);
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
