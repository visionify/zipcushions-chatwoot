<script>
import { mapGetters } from 'vuex';
import { getContrastingTextColor } from '@chatwoot/utils';
import CardButton from 'shared/components/CardButton.vue';

export default {
  components: {
    CardButton,
  },
  props: {
    title: {
      type: String,
      default: '',
    },
    description: {
      type: String,
      default: '',
    },
    mediaUrl: {
      type: String,
      default: '',
    },
    actions: {
      type: Array,
      default: () => [],
    },
  },
  emits: ['select'],
  data() {
    return {
      isSelected: false,
    };
  },
  computed: {
    ...mapGetters({
      widgetColor: 'appConfig/getWidgetColor',
    }),
    textColor() {
      return getContrastingTextColor(this.widgetColor);
    },
    linkActions() {
      return this.actions.filter(function(a) { return a.type !== 'postback'; });
    },
    postbackAction() {
      var found = this.actions.filter(function(a) { return a.type === 'postback'; });
      return found.length > 0 ? found[0] : null;
    },
  },
  methods: {
    onSelect() {
      if (this.isSelected) return;
      this.isSelected = true;
      this.$emit('select', this.postbackAction.payload);
    },
  },
};
</script>

<template>
  <div
    class="carousel-card"
    :class="{ 'carousel-card--selected': isSelected }"
  >
    <img
      v-if="mediaUrl"
      class="carousel-card__image"
      :src="mediaUrl"
      :alt="title"
    />
    <div class="carousel-card__body">
      <h4 class="carousel-card__title">
        {{ title }}
      </h4>
      <p v-if="description" class="carousel-card__desc">
        {{ description }}
      </p>
      <div class="carousel-card__actions">
        <CardButton
          v-for="action in linkActions"
          :key="action.text"
          :action="action"
          class="carousel-card__view-btn"
        />
        <button
          v-if="postbackAction"
          class="carousel-card__select-btn"
          :style="{ background: isSelected ? '#22c55e' : widgetColor }"
          :disabled="isSelected"
          @click="onSelect"
        >
          {{ isSelected ? 'Selected ✓' : postbackAction.text }}
        </button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.carousel-card {
  min-width: 175px;
  max-width: 175px;
  flex-shrink: 0;
  scroll-snap-align: start;
  border-radius: 12px;
  overflow: hidden;
  background: #fff;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.10);
  border: 1px solid rgba(0, 0, 0, 0.06);
  transition: transform 0.2s, box-shadow 0.2s;
}
.carousel-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 14px rgba(0, 0, 0, 0.15);
}
.carousel-card--selected {
  border: 2px solid #1b8ceb;
  box-shadow: 0 2px 12px rgba(27, 140, 235, 0.25);
}

/* Image */
.carousel-card__image {
  width: 100%;
  height: 120px;
  object-fit: cover;
  display: block;
}

/* Body */
.carousel-card__body {
  padding: 10px 12px;
}

/* Title */
.carousel-card__title {
  font-size: 13px;
  font-weight: 600;
  color: #1a1a1a;
  line-height: 1.3;
  margin: 0 0 4px 0;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

/* Description */
.carousel-card__desc {
  font-size: 11px;
  color: #888;
  line-height: 1.4;
  margin: 0 0 8px 0;
  display: -webkit-box;
  -webkit-line-clamp: 1;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

/* Actions container */
.carousel-card__actions {
  display: flex;
  gap: 6px;
}

/* View button override */
.carousel-card__view-btn {
  flex: 1;
}
.carousel-card__view-btn :deep(.action-button) {
  font-size: 11px !important;
  padding: 5px 0 !important;
  border-radius: 8px !important;
  margin-top: 0 !important;
}

/* Select button */
.carousel-card__select-btn {
  flex: 1;
  padding: 5px 0;
  border: none;
  border-radius: 8px;
  color: #fff;
  font-size: 11px;
  font-weight: 600;
  cursor: pointer;
  text-align: center;
  transition: all 0.2s;
}
.carousel-card__select-btn:hover:not(:disabled) {
  opacity: 0.9;
  transform: scale(0.98);
}
.carousel-card__select-btn:disabled {
  cursor: default;
}

/* Dark mode */
:global(.dark) .carousel-card {
  background: #2a2a2a;
  border-color: rgba(255, 255, 255, 0.08);
}
:global(.dark) .carousel-card__title {
  color: #e8e8e8;
}
:global(.dark) .carousel-card__desc {
  color: #999;
}

/* Mobile */
@media (max-width: 400px) {
  .carousel-card {
    min-width: 155px;
    max-width: 155px;
  }
  .carousel-card__image {
    height: 100px;
  }
}
</style>
