<script setup>
import { toRef, computed } from 'vue';
import { useRouter } from 'vue-router';
import { useStore } from 'vuex';
import FluentIcon from 'shared/components/FluentIcon/Index.vue';
import HeaderActions from './HeaderActions.vue';
import AvailabilityContainer from 'widget/components/Availability/AvailabilityContainer.vue';
import { useAvailability } from 'widget/composables/useAvailability';

const props = defineProps({
  avatarUrl: { type: String, default: '' },
  title: { type: String, default: '' },
  showPopoutButton: { type: Boolean, default: false },
  showBackButton: { type: Boolean, default: false },
  availableAgents: { type: Array, default: () => [] },
});

const availableAgents = toRef(props, 'availableAgents');

const router = useRouter();
const store = useStore();
const { isOnline } = useAvailability(availableAgents);

const customAttrs = computed(() => {
  return store.getters['conversationAttributes/getCustomAttributes'] || {};
});

const sessionCode = computed(() => {
  return customAttrs.value.session_code || '';
});

// Dynamic styling — sanitized values only
const sanitizeCss = (val, fallback) => {
  if (!val || typeof val !== 'string') return fallback;
  // Block anything that looks like an injection (urls, expressions, semicolons)
  if (/[;{}()\\]|url|expression|javascript/i.test(val)) return fallback;
  return val.substring(0, 50); // Limit length
};

const sessionCodeStyle = computed(() => {
  var attrs = customAttrs.value;
  return {
    fontSize: sanitizeCss(attrs.session_code_size, '11px'),
    fontWeight: sanitizeCss(attrs.session_code_weight, '500'),
    color: sanitizeCss(attrs.session_code_color, 'rgba(0, 0, 0, 0.4)'),
    letterSpacing: sanitizeCss(attrs.session_code_spacing, '0.5px'),
    fontFamily: sanitizeCss(attrs.session_code_font, 'inherit'),
  };
});

const onBackButtonClick = () => {
  router.replace({ name: 'home' });
};
</script>

<template>
  <header class="flex justify-between w-full p-5 bg-n-background gap-2">
    <div class="flex items-center">
      <button
        v-if="showBackButton"
        class="px-2 ltr:-ml-3 rtl:-mr-3"
        @click="onBackButtonClick"
      >
        <FluentIcon icon="chevron-left" size="24" class="text-n-slate-12" />
      </button>
      <img
        v-if="avatarUrl"
        class="w-8 h-8 ltr:mr-3 rtl:ml-3 rounded-full"
        :src="avatarUrl"
        alt="avatar"
      />
      <div class="flex flex-col gap-1">
        <div
          class="flex items-center text-base font-medium leading-4 text-n-slate-12"
        >
          <span v-dompurify-html="title" class="ltr:mr-1 rtl:ml-1" />
          <span
            v-if="sessionCode"
            class="session-code"
            :style="sessionCodeStyle"
          >
            &middot; {{ sessionCode }}
          </span>
          <div
            :class="`h-2 w-2 rounded-full ltr:ml-1 rtl:mr-1
              ${isOnline ? 'bg-n-teal-10' : 'hidden'}`"
          />
        </div>
        <AvailabilityContainer
          :agents="availableAgents"
          :show-header="false"
          :show-avatars="false"
          text-classes="text-xs leading-3"
        />
      </div>
    </div>
    <HeaderActions :show-popout-button="showPopoutButton" />
  </header>
</template>

<style scoped>
.session-code {
  white-space: nowrap;
}
</style>
