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
      swipeHintTimer: null,
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
    cardsPerPage() {
      return 1;
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
      this.swipeHintTimer = setTimeout(() => {
        this.showSwipeHint = false;
      }, 5000);

      this.$nextTick(() => {
        var viewport = this.$refs.cardsViewport;
        if (viewport) {
          viewport.addEventListener('touchstart', this.handleTouchStart, { passive: true });
          viewport.addEventListener('touchmove', this.handleTouchMove, { passive: false });
          viewport.addEventListener('touchend', this.handleTouchEnd, { passive: false });
        }
      });
    }
  },
  beforeUnmount() {
    if (this.swipeHintTimer) {
      clearTimeout(this.swipeHintTimer);
      this.swipeHintTimer = null;
    }
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
      <div class="carousel-container">
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
      </div>

      <!-- Nav row: prev arrow, page dots, next arrow -->
      <div v-if="hasMultipleCards" class="carousel-nav">
        <button
          class="carousel-arrow carousel-arrow-left"
          :class="{ disabled: !canGoPrev }"
          @click="prevPage"
        >
          ‹
        </button>
        <div class="carousel-dots">
          <span
            v-for="page in totalPages"
            :key="'page-' + page"
            class="carousel-dot"
            :class="{ active: (page - 1) === currentPage }"
            @click="goToPage(page - 1)"
          />
        </div>
        <button
          class="carousel-arrow carousel-arrow-right"
          :class="{ disabled: !canGoNext }"
          @click="nextPage"
        >
          ›
        </button>
      </div>

      <div v-if="hasMultipleCards && showSwipeHint" class="swipe-hint">
        <span class="swipe-hint-text">← swipe or tap arrows →</span>
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
/* ===== OVERFLOW KILL — every level locked ===== */
.chat-bubble-wrap {
  overflow-x: hidden !important;
  overflow-y: visible;
  max-width: 100% !important;
  width: 100%;
}

.carousel-wrapper {
  width: 100%;
  max-width: 100%;
  padding: 4px 0;
  overflow: visible !important;
}

.carousel-container {
  position: relative;
  width: 100%;
  max-width: 100%;
  overflow: visible !important;
}

/* ===== VIEWPORT — clips everything ===== */
.cards-viewport {
  overflow: hidden !important;
  width: 100% !important;
  max-width: 100% !important;
  touch-action: none;
  box-sizing: border-box;
}

/* ===== TRACK — flex row, NO width set ===== */
.cards-track {
  display: flex;
  will-change: transform;
}

/* ===== CARD = exactly one viewport per slide ===== */
.card-slide {
  flex: 0 0 100%;
  width: 100%;
  box-sizing: border-box;
  overflow: hidden;
}

/* ===== FORCE ChatCard internals to obey container ===== */
.card-slide :deep(*) {
  max-width: 100% !important;
  box-sizing: border-box !important;
}

.card-slide :deep(.chat-card),
.card-slide :deep(.carousel-card) {
  width: 100% !important;
  min-width: 0 !important;
  max-width: 100% !important;
  overflow: hidden !important;
  word-wrap: break-word !important;
  overflow-wrap: break-word !important;
}

.card-slide :deep(img),
.card-slide :deep(video),
.card-slide :deep(.carousel-card__image) {
  width: 100% !important;
  max-width: 100% !important;
  height: 140px !important;
  object-fit: cover;
  display: block;
  border-radius: 10px !important;
}

.card-slide :deep(h4),
.card-slide :deep(h5),
.card-slide :deep(.title) {
  font-size: 12px !important;
  line-height: 1.3 !important;
  overflow: hidden !important;
  text-overflow: ellipsis !important;
  display: -webkit-box !important;
  -webkit-line-clamp: 2 !important;
  -webkit-box-orient: vertical !important;
  word-wrap: break-word !important;
  margin: 2px 0 !important;
}

.card-slide :deep(p),
.card-slide :deep(.description) {
  font-size: 10px !important;
  line-height: 1.2 !important;
  overflow: hidden !important;
  text-overflow: ellipsis !important;
  display: -webkit-box !important;
  -webkit-line-clamp: 2 !important;
  -webkit-box-orient: vertical !important;
  word-wrap: break-word !important;
  margin: 2px 0 !important;
}

.card-slide :deep(a),
.card-slide :deep(button) {
  font-size: 11px !important;
  padding: 5px 6px !important;
  max-width: 100% !important;
  overflow: hidden !important;
  text-overflow: ellipsis !important;
  white-space: nowrap !important;
  box-sizing: border-box !important;
}

/* ===== NAV ROW: arrows + dots inline below the card ===== */
.carousel-nav {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 12px;
  padding: 8px 0 4px 0;
}

.carousel-arrow {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  border: none;
  background: #4a4a4a;
  font-size: 16px;
  font-weight: bold;
  cursor: pointer;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.25);
  display: flex;
  align-items: center;
  justify-content: center;
  color: #ffffff;
  transition: all 0.2s;
  user-select: none;
  padding: 0;
  line-height: 1;
  flex-shrink: 0;
}
.carousel-arrow:hover {
  background: #2a2a2a;
  box-shadow: 0 3px 10px rgba(0, 0, 0, 0.35);
}
.carousel-arrow:active {
  transform: scale(0.92);
}
.carousel-arrow.disabled {
  opacity: 0.3;
  pointer-events: none;
}

/* ===== PAGE DOTS (inside carousel-nav) ===== */
.carousel-dots {
  display: flex;
  gap: 5px;
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

/* ===== SWIPE HINT ===== */
.swipe-hint {
  text-align: center;
  padding: 4px 0 2px 0;
  animation: fadeHint 5s ease forwards;
}
.swipe-hint-text {
  font-size: 10px;
  color: #aaa;
  letter-spacing: 0.3px;
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

/* ===== DARK MODE ===== */
:global(.dark) .carousel-arrow {
  background: #2a2a2a;
  color: #ffffff;
}
:global(.dark) .carousel-arrow:hover {
  background: #1a1a1a;
}
:global(.dark) .carousel-dot {
  background: #444;
}
</style>
