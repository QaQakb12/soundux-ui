<template>
  <div class="cf-sound" :class="[branch ? 'cf-sound--' + branch : '', { 'cf-sound--fired': fired, 'cf-sound--code-only': !parsed.code }]" :data-sound-id="sound.id" @contextmenu="onContext">
    <span
      class="cf-sound__handle"
      role="button"
      tabindex="0"
      :aria-label="'Move ' + sound.name"
      title="Drag to move; use Alt + arrow keys when focused"
    >
      <svg class="cf-icon" width="16" height="16" viewBox="0 0 24 24" fill="currentColor" aria-hidden="true">
        <circle cx="9" cy="6" r="1.7" />
        <circle cx="15" cy="6" r="1.7" />
        <circle cx="9" cy="12" r="1.7" />
        <circle cx="15" cy="12" r="1.7" />
        <circle cx="9" cy="18" r="1.7" />
        <circle cx="15" cy="18" r="1.7" />
      </svg>
    </span>
    <button
      class="cf-sound__main"
      type="button"
      :id="'sound-' + sound.id"
      :aria-label="'Play ' + sound.name"
      :title="sound.name"
      @click="fire"
    >
      <span v-if="parsed.code" class="cf-sound__code">{{ parsed.code }}</span>
      <span class="cf-sound__label">{{ parsed.label }}</span>
    </button>
    <span class="cf-sound__meta">
      <span v-if="branch" class="cf-sound__branch">{{ branch === 'yes' ? 'YES' : 'NO' }}</span>
      <span v-if="sound.isFavorite" class="cf-sound__fav" title="Favorite">★</span>
      <span v-if="hasHotkey" class="cf-sound__hotkey" title="Has a hotkey">⌨</span>
    </span>
  </div>
</template>

<script lang="ts">
import Vue, { PropType } from 'vue';
import { Sound } from '@/types';

export default Vue.extend({
  name: 'CallFlowSoundButton',
  props: {
    sound: {
      type: Object as PropType<Sound>,
      required: true,
    },
    context: {
      type: Function as PropType<(event: MouseEvent) => void>,
      default: null,
    },
  },
  data() {
    return {
      fired: false,
      firedTimer: 0 as number,
    };
  },
  computed: {
    parsed(): { code: string; label: string } {
      const match = /^\s*(\d{1,2}[A-Za-z]?)(?=$|[\s._:)\-–—])[\s._:)\-–—]*(.*)$/.exec(this.sound.name);
      if (match && match[2].trim()) {
        return { code: match[1].toUpperCase(), label: match[2].trim() };
      }
      return { code: '', label: this.sound.name.trim() };
    },
    branch(): string | null {
      if (/(^|[^a-z])(yes|oui)([^a-z]|$)/i.test(this.sound.name)) {
        return 'yes';
      }
      if (/(^|[^a-z])(no|non)([^a-z]|$)/i.test(this.sound.name)) {
        return 'no';
      }
      return null;
    },
    hasHotkey(): boolean {
      return Array.isArray(this.sound.hotkeys) && this.sound.hotkeys.length > 0;
    },
  },
  beforeDestroy() {
    window.clearTimeout(this.firedTimer);
  },
  methods: {
    onContext(event: MouseEvent): void {
      if (this.context) {
        this.context(event);
      }
    },
    fire(): void {
      this.$store.dispatch('playSound', this.sound);
      this.fired = false;
      window.clearTimeout(this.firedTimer);
      this.$nextTick(() => {
        this.fired = true;
        this.firedTimer = window.setTimeout(() => {
          this.fired = false;
        }, 420);
      });
    },
  },
});
</script>
