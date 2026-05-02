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
          class="carousel-card__link-btn"
        />
        <button
          v-if="postbackAction"
          class="carousel-card__select-btn"
          :style="{ background: isSelected ? '#22c55e' : widgetColor }"
          :disabled="isSelected"
          @click="onSelect"
        >
          {{ isSelected ? 'Selected' : postbackAction.text }}
        </button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.carousel-card {
  width: 100%;
  min-width: 0;
  border-radius: 14px;
  overflow: hidden;
  background: #ffffff;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  border: 1.5px solid #e0e0e0;
  padding: 10px;
  transition: transform 0.2s, box-shadow 0.2s;
  box-sizing: border-box;
}
.carousel-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 14px rgba(0, 0, 0, 0.15);
}
.carousel-card--selected {
  border: 2px solid #22c55e;
  box-shadow: 0 2px 12px rgba(34, 197, 94, 0.25);
}

/* Image — framed inside the card with rounded corners */
.carousel-card__image {
  width: 100%;
  height: 140px;
  object-fit: cover;
  display: block;
  border-radius: 10px;
}

/* Body — tight padding since card already has 10px around */
.carousel-card__body {
  padding: 10px 4px 4px 4px;
}

/* Title */
.carousel-card__title {
  font-size: 14px;
  font-weight: 600;
  color: #1a1a1a;
  line-height: 1.3;
  margin: 0 0 4px 0;
  word-wrap: break-word;
  overflow-wrap: break-word;
}

/* Description */
.carousel-card__desc {
  font-size: 12px;
  color: #888;
  line-height: 1.3;
  margin: 0 0 8px 0;
  word-wrap: break-word;
  overflow-wrap: break-word;
}

/* Actions — stacked vertically */
.carousel-card__actions {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

/* View Fabric button */
.carousel-card__link-btn {
  width: 100%;
}
.carousel-card__link-btn :deep(.action-button) {
  width: 100% !important;
  font-size: 12px !important;
  padding: 8px 12px !important;
  border-radius: 8px !important;
  margin-top: 0 !important;
  text-align: center !important;
  box-sizing: border-box !important;
}

/* Select button */
.carousel-card__select-btn {
  width: 100%;
  padding: 8px 12px;
  border: none;
  border-radius: 8px;
  color: #fff;
  font-size: 12px;
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
  opacity: 0.9;
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
</style>
