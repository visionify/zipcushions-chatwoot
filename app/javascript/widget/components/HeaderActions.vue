<script>
import { mapGetters } from 'vuex';
import { IFrameHelper, RNHelper } from 'widget/helpers/utils';
import { popoutChatWindow } from '../helpers/popoutHelper';
import FluentIcon from 'shared/components/FluentIcon/Index.vue';
import configMixin from 'widget/mixins/configMixin';
import { CONVERSATION_STATUS } from 'shared/constants/messages';

export default {
  name: 'HeaderActions',
  components: { FluentIcon },
  mixins: [configMixin],
  props: {
    showPopoutButton: {
      type: Boolean,
      default: false,
    },
    showEndConversationButton: {
      type: Boolean,
      default: true,
    },
  },
  data() {
    return {
      showQRDialog: false,
      copied: false,
    };
  },
  computed: {
    ...mapGetters({
      conversationAttributes: 'conversationAttributes/getConversationParams',
      canUserEndConversation: 'appConfig/getCanUserEndConversation',
      sessionCode: 'conversationAttributes/getSessionCode',
    }),
    canLeaveConversation() {
      return [
        CONVERSATION_STATUS.OPEN,
        CONVERSATION_STATUS.SNOOZED,
        CONVERSATION_STATUS.PENDING,
      ].includes(this.conversationStatus);
    },
    isIframe() {
      return IFrameHelper.isIFrame();
    },
    isRNWebView() {
      return RNHelper.isRNWebView();
    },
    showHeaderActions() {
      return this.isIframe || this.isRNWebView || this.hasWidgetOptions;
    },
    conversationStatus() {
      return this.conversationAttributes.status;
    },
    hasWidgetOptions() {
      return this.showPopoutButton || this.conversationStatus === 'open';
    },
    qrCodeUrl() {
      if (!this.sessionCode) return '';
      var resumeUrl = 'https://zipcushions.com/pages/owlee-replacement-cushions?session=' + this.sessionCode;
      return 'https://api.qrserver.com/v1/create-qr-code/?size=180x180&data=' + encodeURIComponent(resumeUrl) + '&bgcolor=f8f8f8';
    },
  },
  methods: {
    popoutWindow() {
      this.closeWindow();
      const {
        location: { origin },
        chatwootWebChannel: { websiteToken },
        authToken,
      } = window;
      popoutChatWindow(
        origin,
        websiteToken,
        this.$root.$i18n.locale,
        authToken
      );
    },
    closeWindow() {
      if (IFrameHelper.isIFrame()) {
        IFrameHelper.sendMessage({ event: 'closeWindow' });
      } else if (RNHelper.isRNWebView) {
        RNHelper.sendMessage({ type: 'close-widget' });
      }
    },
    resolveConversation() {
      this.$store.dispatch('conversation/resolveConversation');
    },
    openQRDialog() {
      this.showQRDialog = true;
      this.copied = false;
    },
    closeQRDialog() {
      this.showQRDialog = false;
    },
    copySessionCode() {
      if (!this.sessionCode) return;
      navigator.clipboard.writeText(this.sessionCode).then(() => {
        this.copied = true;
        setTimeout(() => { this.copied = false; }, 1500);
      }).catch(() => {
        // Fallback for older browsers
        var el = document.createElement('textarea');
        el.value = this.sessionCode;
        document.body.appendChild(el);
        el.select();
        document.execCommand('copy');
        document.body.removeChild(el);
        this.copied = true;
        setTimeout(() => { this.copied = false; }, 1500);
      });
    },
  },
};
</script>

<!-- eslint-disable-next-line vue/no-root-v-if -->
<template>
  <div v-if="showHeaderActions" class="actions flex items-center gap-2">
    <!-- QR Button — only show when session code exists -->
    <button
      v-if="sessionCode"
      class="button transparent compact qr-button"
      title="Save your session"
      @click="openQRDialog"
    >
      <svg width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round" class="text-n-slate-12 qr-icon">
        <rect x="3" y="3" width="7" height="7" rx="1.5" />
        <rect x="14" y="3" width="7" height="7" rx="1.5" />
        <rect x="3" y="14" width="7" height="7" rx="1.5" />
        <rect x="6" y="6" width="1" height="1" fill="currentColor" stroke="none" />
        <rect x="17" y="6" width="1" height="1" fill="currentColor" stroke="none" />
        <rect x="6" y="17" width="1" height="1" fill="currentColor" stroke="none" />
        <path d="M14 14h3v3h-3z" />
        <path d="M18 14h3" />
        <path d="M14 18v3" />
        <path d="M18 18h3v3" />
      </svg>
    </button>

    <button
      v-if="
        canLeaveConversation &&
        canUserEndConversation &&
        hasEndConversationEnabled &&
        showEndConversationButton
      "
      class="button transparent compact"
      :title="$t('END_CONVERSATION')"
      @click="resolveConversation"
    >
      <FluentIcon icon="sign-out" size="22" class="text-n-slate-12" />
    </button>
    <button
      v-if="showPopoutButton"
      class="button transparent compact new-window--button"
      @click="popoutWindow"
    >
      <FluentIcon icon="open" size="22" class="text-n-slate-12" />
    </button>
    <button
      class="button transparent compact close-button"
      :class="{
        'rn-close-button': isRNWebView,
      }"
      @click="closeWindow"
    >
      <FluentIcon icon="dismiss" size="24" class="text-n-slate-12" />
    </button>

    <!-- QR Dialog Overlay -->
    <div
      v-if="showQRDialog"
      class="qr-overlay"
      @click.self="closeQRDialog"
    >
      <div class="qr-dialog">
        <!-- Logo + Brand -->
        <img
          class="qr-dialog__logo-top"
          src="https://cdn.shopify.com/s/files/1/0614/7453/7714/files/ZIPCushions-logo.webp?v=1771313144"
          alt="ZIPCushions"
        />
        <div class="qr-dialog__brand">ZIPCushions</div>
        <div class="qr-dialog__subtitle">Save your conversation</div>

        <!-- QR code with logo watermark -->
        <div class="qr-dialog__qr-wrap">
          <img
            class="qr-dialog__logo-watermark"
            src="https://cdn.shopify.com/s/files/1/0614/7453/7714/files/ZIPCushions-logo.webp?v=1771313144"
            alt=""
          />
          <img
            v-if="qrCodeUrl"
            class="qr-dialog__qr-img"
            :src="qrCodeUrl"
            width="160"
            height="160"
            alt="Session QR Code"
          />
        </div>

        <!-- Session code -->
        <div class="qr-dialog__code-label">Session Code</div>
        <div class="qr-dialog__code-wrap">
          <span class="qr-dialog__code">{{ sessionCode }}</span>
          <button
            class="qr-dialog__copy-btn"
            :class="{ 'qr-dialog__copy-btn--copied': copied }"
            @click="copySessionCode"
          >
            {{ copied ? 'Copied!' : 'Copy' }}
          </button>
        </div>

        <!-- Helper text -->
        <div class="qr-dialog__helper">
          Scan the QR or type this code to pick up<br />where you left off, on any device.
        </div>

        <button class="qr-dialog__close-btn" @click="closeQRDialog">
          Close
        </button>
      </div>
    </div>
  </div>
</template>

<style scoped lang="scss">
.actions {
  .close-button {
    display: none;
  }

  .rn-close-button {
    display: block !important;
  }
}

.qr-button {
  opacity: 0.85;
  transition: all 0.15s ease;
  padding: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  &:hover {
    opacity: 1;
  }
}

.qr-icon {
  display: block;
}

/* QR Dialog Overlay */
.qr-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(15, 20, 30, 0.6);
  backdrop-filter: blur(8px);
  -webkit-backdrop-filter: blur(8px);
  z-index: 9999;
  display: flex;
  align-items: center;
  justify-content: center;
  animation: overlayFadeIn 0.25s ease;
}

@keyframes overlayFadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

@keyframes dialogSlideIn {
  from { opacity: 0; transform: translateY(12px); }
  to { opacity: 1; transform: translateY(0); }
}

.qr-dialog {
  background: linear-gradient(145deg, #ffffff, #f8fafb);
  border-radius: 24px;
  padding: 32px 28px 24px;
  text-align: center;
  max-width: 300px;
  width: 92%;
  box-shadow: 0 32px 64px rgba(0, 0, 0, 0.12), 0 0 0 1px rgba(0, 0, 0, 0.04);
  animation: dialogSlideIn 0.3s cubic-bezier(0.16, 1, 0.3, 1);
}

/* Logo at top */
.qr-dialog__logo-top {
  width: 36px;
  height: 36px;
  margin: 0 auto 6px;
  object-fit: contain;
}

/* Brand name */
.qr-dialog__brand {
  font-size: 15px;
  font-weight: 700;
  color: #2d3748;
  letter-spacing: 0.5px;
  margin-bottom: 2px;
}

/* Subtitle */
.qr-dialog__subtitle {
  font-size: 11px;
  color: #a0aec0;
  margin-bottom: 20px;
  font-weight: 400;
}

/* QR code wrapper */
.qr-dialog__qr-wrap {
  position: relative;
  background: #fff;
  border: 1px solid #edf2f7;
  border-radius: 16px;
  padding: 16px;
  margin-bottom: 20px;
  display: flex;
  justify-content: center;
  align-items: center;
}

.qr-dialog__logo-watermark {
  position: absolute;
  width: 36px;
  height: 36px;
  object-fit: contain;
  z-index: 2;
  pointer-events: none;
  background: #fff;
  border-radius: 50%;
  padding: 4px;
  box-shadow: 0 0 0 3px #fff;
}

.qr-dialog__qr-img {
  display: block;
  position: relative;
  z-index: 1;
  border-radius: 4px;
}

/* Session code section */
.qr-dialog__code-label {
  font-size: 10px;
  font-weight: 600;
  color: #a0aec0;
  text-transform: uppercase;
  letter-spacing: 1.2px;
  margin-bottom: 8px;
}

.qr-dialog__code-wrap {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
  background: #f7fafc;
  border: 1px solid #e2e8f0;
  border-radius: 12px;
  padding: 12px 16px;
  margin-bottom: 12px;
}

.qr-dialog__code {
  font-family: 'SF Mono', 'Menlo', 'Consolas', monospace;
  font-size: 18px;
  font-weight: 700;
  color: #2b6cb0;
  letter-spacing: 2.5px;
}

.qr-dialog__copy-btn {
  background: #2b6cb0;
  color: #fff;
  border: none;
  border-radius: 8px;
  padding: 6px 14px;
  cursor: pointer;
  font-size: 11px;
  font-weight: 600;
  transition: all 0.15s ease;
  letter-spacing: 0.3px;

  &:hover {
    background: #2c5282;
    transform: translateY(-1px);
  }

  &--copied {
    background: #38a169;
  }
}

/* Helper text */
.qr-dialog__helper {
  font-size: 11px;
  color: #a0aec0;
  line-height: 1.5;
  margin-bottom: 18px;
}

/* Close button */
.qr-dialog__close-btn {
  background: transparent;
  color: #a0aec0;
  border: 1px solid #e2e8f0;
  border-radius: 12px;
  padding: 9px 32px;
  cursor: pointer;
  font-size: 12px;
  font-weight: 500;
  transition: all 0.15s ease;
  letter-spacing: 0.3px;

  &:hover {
    background: #f7fafc;
    color: #4a5568;
    border-color: #cbd5e0;
  }
}

/* Dark mode */
:global(.dark) .qr-dialog {
  background: linear-gradient(145deg, #1a202c, #171923);
  box-shadow: 0 32px 64px rgba(0, 0, 0, 0.4);
}
:global(.dark) .qr-dialog__brand {
  color: #e2e8f0;
}
:global(.dark) .qr-dialog__subtitle,
:global(.dark) .qr-dialog__code-label {
  color: #718096;
}
:global(.dark) .qr-dialog__qr-wrap {
  background: #fff;
  border-color: #e2e8f0;
}
:global(.dark) .qr-dialog__code-wrap {
  background: #2d3748;
  border-color: #4a5568;
}
:global(.dark) .qr-dialog__code {
  color: #63b3ed;
}
:global(.dark) .qr-dialog__helper {
  color: #718096;
}
:global(.dark) .qr-dialog__close-btn {
  border-color: #4a5568;
  color: #718096;
  &:hover {
    background: #2d3748;
    color: #a0aec0;
  }
}
</style>
