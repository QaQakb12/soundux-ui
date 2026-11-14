<template>
  <section class="cf-col" :class="['cf-col--' + group.id, { 'cf-col--tray': variant === 'tray' }]">
    <header class="cf-col__head">
      <span v-if="index" class="cf-col__index">{{ index }}</span>
      <div v-if="editing" class="cf-edit">
        <input
          ref="input"
          v-model="draft"
          class="cf-edit__input"
          type="text"
          maxlength="28"
          :aria-label="'Rename ' + group.title"
          @keydown.enter.prevent.stop="save"
          @keydown.esc.prevent.stop="cancel"
          @blur="onBlur"
        />
        <button class="cf-edit__btn cf-edit__btn--ok" type="button" title="Save (Enter)" aria-label="Save title" @mousedown.prevent @click="save">✓</button>
        <button class="cf-edit__btn" type="button" title="Cancel (Esc)" aria-label="Cancel renaming" @mousedown.prevent @click="cancel">×</button>
      </div>
      <div v-else class="cf-col__titlewrap">
        <button class="cf-title" type="button" :title="'Rename ' + group.title" @click="startEdit">
          <span class="cf-title__text">{{ group.title }}</span>
          <span class="cf-title__icon" aria-hidden="true">✎</span>
        </button>
      </div>
      <span class="cf-col__count" :aria-label="sounds.length + ' recordings'">{{ sounds.length }}</span>
    </header>

    <draggable
      v-model="mutableSounds"
      class="cf-list"
      :group="dragGroup"
      handle=".cf-sound__handle"
      :animation="160"
      :force-fallback="false"
      :fallback-on-body="true"
      :fallback-tolerance="4"
      :empty-insert-threshold="48"
      :scroll="true"
      :scroll-sensitivity="60"
      :scroll-speed="12"
      ghost-class="cf-ghost"
      chosen-class="cf-chosen"
      drag-class="cf-drag"
      @start="$emit('drag-start')"
      @change="$emit('commit')"
      @end="$emit('drag-end')"
    >
      <SoundContextMenu v-for="sound in mutableSounds" :key="sound.id" :sound="sound">
        <template #default="{ context }">
          <CallFlowSoundButton :sound="sound" :context="context" />
        </template>
      </SoundContextMenu>
    </draggable>
  </section>
</template>

<script lang="ts">
import Vue, { PropType } from 'vue';
import draggable from 'vuedraggable';
import { Sound } from '@/types';
import SoundContextMenu from '@/components/SoundContextMenu.vue';
import CallFlowSoundButton from '@/components/callflow/CallFlowSoundButton.vue';

interface CallFlowGroup {
  id: string;
  title: string;
}

export default Vue.extend({
  name: 'CallFlowColumn',
  components: {
    draggable,
    SoundContextMenu,
    CallFlowSoundButton,
  },
  props: {
    group: {
      type: Object as PropType<CallFlowGroup>,
      required: true,
    },
    sounds: {
      type: Array as PropType<Sound[]>,
      required: true,
    },
    index: {
      type: Number,
      default: 0,
    },
    variant: {
      type: String,
      default: 'column',
    },
  },
  data() {
    return {
      editing: false,
      draft: '',
    };
  },
  computed: {
    mutableSounds: {
      get(): Sound[] {
        return this.sounds;
      },
      set(value: Sound[]) {
        this.$emit('update:sounds', value);
      },
    },
    dragGroup(): { name: string; pull: boolean; put: boolean } {
      return { name: 'callflow-sounds', pull: true, put: true };
    },
  },
  methods: {
    startEdit(): void {
      this.draft = this.group.title;
      this.editing = true;
      this.$nextTick(() => {
        const input = this.$refs.input as HTMLInputElement | undefined;
        if (input) {
          input.focus();
          input.select();
        }
      });
    },
    save(): void {
      if (!this.editing) {
        return;
      }
      const title = this.draft.replace(/\s+/g, ' ').trim().slice(0, 28);
      this.editing = false;
      this.$emit('rename', this.group.id, title || this.group.title);
      this.restoreFocus();
    },
    cancel(): void {
      this.editing = false;
      this.restoreFocus();
    },
    onBlur(): void {
      this.save();
    },
    restoreFocus(): void {
      this.$nextTick(() => {
        const title = this.$el.querySelector('.cf-title') as HTMLElement | null;
        if (title) {
          title.focus();
        }
      });
    },
  },
});
</script>
