<template>
  <div class="callflow" :class="{ 'callflow--dragging': dragging }" id="grid-view">
    <div class="callflow__columns">
      <CallFlowColumn
        v-for="(groupId, index) in mainGroupIds"
        :key="groupId"
        :group="{ id: groupId, title: titles[groupId] }"
        :sounds="columns[groupId]"
        :index="index + 1"
        @update:sounds="setColumn(groupId, $event)"
        @rename="renameGroup"
        @commit="scheduleSave"
        @drag-start="dragging = true"
        @drag-end="onDragEnd"
      />
    </div>

    <CallFlowColumn
      class="callflow__tray"
      variant="tray"
      :group="{ id: 'other', title: titles.other }"
      :sounds="columns.other"
      @update:sounds="setColumn('other', $event)"
      @rename="renameGroup"
      @commit="scheduleSave"
      @drag-start="dragging = true"
      @drag-end="onDragEnd"
    />

    <footer class="callflow__foot">
      <span class="callflow__status" :class="{ 'callflow__status--warn': saveError }" :title="statusTitle">
        <span class="callflow__dot" aria-hidden="true"></span>
        {{ statusText }}
      </span>
      <span class="callflow__hint">Focus a recording or its grip, then use Alt + arrow keys to move it</span>
      <button class="callflow__reset" type="button" @click="onResetClick">
        <span aria-hidden="true">↺</span>
        {{ confirmReset ? 'Click again to confirm' : 'Reset layout' }}
      </button>
    </footer>
  </div>
</template>

<script lang="ts">
import Vue, { PropType } from 'vue';
import { Sound, Tab } from '@/types';
import CallFlowColumn from '@/components/callflow/CallFlowColumn.vue';

type GroupId = 'opening' | 'qualification' | 'closing' | 'other';

interface LayoutGroup {
  id: GroupId;
  title: string;
  soundIds: number[];
}

interface StoredLayout {
  version: number;
  tabPath: string;
  groups: LayoutGroup[];
}

interface LayoutStore {
  [key: string]: StoredLayout;
}

type Columns = Record<GroupId, Sound[]>;

const MAIN_GROUPS: GroupId[] = ['opening', 'qualification', 'closing'];
const ALL_GROUPS: GroupId[] = ['opening', 'qualification', 'closing', 'other'];
const DEFAULT_TITLES: Record<GroupId, string> = {
  opening: 'Opening',
  qualification: 'Qualification',
  closing: 'Closing',
  other: 'Unsorted',
};

function emptyColumns(): Columns {
  return { opening: [], qualification: [], closing: [], other: [] };
}

function classifySound(name: string): GroupId {
  const match = /^\s*(\d{1,2})(?!\d)/.exec(name);
  if (!match) {
    return 'other';
  }
  const number = parseInt(match[1], 10);
  if (number >= 2 && number <= 4) {
    return 'opening';
  }
  if (number >= 5 && number <= 8) {
    return 'qualification';
  }
  if (number >= 9 && number <= 10) {
    return 'closing';
  }
  return 'other';
}

function isObject(value: any): value is Record<string, any> {
  return !!value && typeof value === 'object' && !Array.isArray(value);
}

export default Vue.extend({
  name: 'CallFlowDashboard',
  components: { CallFlowColumn },
  props: {
    tab: {
      type: Object as PropType<Tab>,
      required: true,
    },
  },
  data() {
    return {
      mainGroupIds: MAIN_GROUPS,
      columns: emptyColumns() as Columns,
      titles: { ...DEFAULT_TITLES } as Record<GroupId, string>,
      dragging: false,
      confirmReset: false,
      confirmTimer: 0 as number,
      saveTimer: 0 as number,
      ready: false,
      isPermanent: false,
      saving: false,
      saveError: false,
      storage: {} as LayoutStore,
      saveChain: Promise.resolve() as Promise<void>,
    };
  },
  computed: {
    sounds(): Sound[] {
      return this.tab.sounds || [];
    },
    tabKey(): string {
      const path = (this.tab.path || this.tab.name || String(this.tab.id)).replace(/\\/g, '/').replace(/\/+$/, '');
      return this.$store.getters.isLinux ? path : path.toLowerCase();
    },
    statusText(): string {
      if (this.saveError) {
        return 'Could not save layout';
      }
      if (this.saving) {
        return 'Saving layout…';
      }
      return this.isPermanent ? 'Layout saved' : 'Layout is session-only';
    },
    statusTitle(): string {
      return this.isPermanent
        ? 'The Call Flow layout is saved in Soundux configuration and survives restarts.'
        : 'Persistent layout storage is not connected.';
    },
  },
  watch: {
    sounds() {
      if (this.ready) {
        this.reconcileCurrentSounds(true);
      }
    },
    tabKey() {
      this.hydrate();
    },
  },
  mounted() {
    document.addEventListener('keydown', this.handleKeydown, true);
    this.hydrate();
  },
  beforeDestroy() {
    document.removeEventListener('keydown', this.handleKeydown, true);
    window.clearTimeout(this.saveTimer);
    window.clearTimeout(this.confirmTimer);
    this.flushSave();
  },
  methods: {
    async hydrate(): Promise<void> {
      this.ready = false;
      this.isPermanent = false;
      this.saveError = false;
      this.storage = {};
      this.titles = { ...DEFAULT_TITLES };

      try {
        if (typeof window.getCallFlowLayouts === 'function') {
          const raw = await window.getCallFlowLayouts();
          const parsed = JSON.parse(raw || '{}');
          if (isObject(parsed)) {
            this.storage = parsed as LayoutStore;
          }
          this.isPermanent = true;
        }
      } catch (error) {
        console.error('Call Flow: could not load saved layouts', error);
        this.saveError = true;
      }

      const saved = this.storage[this.tabKey];
      this.columns = emptyColumns();
      if (saved && Array.isArray(saved.groups)) {
        saved.groups.forEach(group => {
          if (ALL_GROUPS.indexOf(group.id) < 0) {
            return;
          }
          if (typeof group.title === 'string' && group.title.trim()) {
            this.$set(this.titles, group.id, group.title.trim().slice(0, 28));
          }
          const ids = Array.isArray(group.soundIds) ? group.soundIds : [];
          const byId = new Map<number, Sound>(this.sounds.map(sound => [sound.id, sound]));
          this.$set(
            this.columns,
            group.id,
            ids.map(id => byId.get(id)).filter((sound): sound is Sound => !!sound)
          );
        });
      }

      this.reconcileCurrentSounds(false);
      this.ready = true;
    },
    reconcileCurrentSounds(saveWhenChanged: boolean): void {
      const before = ALL_GROUPS.map(id => this.columns[id].map(sound => sound.id).join(',')).join('|');
      const byId = new Map<number, Sound>(this.sounds.map(sound => [sound.id, sound]));
      const next = emptyColumns();
      const used = new Set<number>();

      ALL_GROUPS.forEach(groupId => {
        (this.columns[groupId] || []).forEach(sound => {
          const current = byId.get(sound.id);
          if (current && !used.has(current.id)) {
            next[groupId].push(current);
            used.add(current.id);
          }
        });
      });

      this.sounds.forEach(sound => {
        if (!used.has(sound.id)) {
          next[classifySound(sound.name)].push(sound);
          used.add(sound.id);
        }
      });

      const after = ALL_GROUPS.map(id => next[id].map(sound => sound.id).join(',')).join('|');
      this.columns = next;
      if (saveWhenChanged && before !== after) {
        this.scheduleSave();
      }
    },
    setColumn(groupId: GroupId, sounds: Sound[]): void {
      this.$set(this.columns, groupId, sounds);
    },
    renameGroup(groupId: GroupId, title: string): void {
      const safeTitle = title.replace(/\s+/g, ' ').trim().slice(0, 28);
      if (safeTitle && safeTitle !== this.titles[groupId]) {
        this.$set(this.titles, groupId, safeTitle);
        this.scheduleSave();
      }
    },
    currentLayout(): StoredLayout {
      return {
        version: 2,
        tabPath: this.tabKey,
        groups: ALL_GROUPS.map(id => ({
          id,
          title: this.titles[id],
          soundIds: this.columns[id].map(sound => sound.id),
        })),
      };
    },
    scheduleSave(): void {
      if (!this.ready || !this.isPermanent) {
        return;
      }
      window.clearTimeout(this.saveTimer);
      this.saveTimer = window.setTimeout(() => this.flushSave(), 300);
    },
    flushSave(): void {
      if (!this.ready || !this.isPermanent || !this.tabKey) {
        return;
      }
      window.clearTimeout(this.saveTimer);
      this.saveTimer = 0;
      this.storage[this.tabKey] = this.currentLayout();
      const payload = JSON.stringify(this.storage);
      this.saveChain = this.saveChain.then(async () => {
        try {
          this.saving = true;
          const ok = await window.saveCallFlowLayouts(payload);
          this.saveError = !ok;
        } catch (error) {
          console.error('Call Flow: could not save layout', error);
          this.saveError = true;
        } finally {
          this.saving = false;
        }
      });
    },
    onDragEnd(): void {
      this.dragging = false;
      this.scheduleSave();
    },
    onResetClick(): void {
      window.clearTimeout(this.confirmTimer);
      if (!this.confirmReset) {
        this.confirmReset = true;
        this.confirmTimer = window.setTimeout(() => {
          this.confirmReset = false;
        }, 3000);
        return;
      }
      this.confirmReset = false;
      this.columns = emptyColumns();
      this.sounds.forEach(sound => this.columns[classifySound(sound.name)].push(sound));
      this.titles = { ...DEFAULT_TITLES };
      this.scheduleSave();
    },
    handleKeydown(event: KeyboardEvent): void {
      if (!event.altKey || event.ctrlKey || event.metaKey || event.shiftKey) {
        return;
      }
      const directions: Record<string, string> = {
        ArrowUp: 'up',
        ArrowDown: 'down',
        ArrowLeft: 'left',
        ArrowRight: 'right',
      };
      const direction = directions[event.key];
      if (!direction) {
        return;
      }
      const target = event.target as HTMLElement | null;
      if (!target || target.closest('.cf-edit')) {
        return;
      }
      const soundElement = target.closest('.cf-sound') as HTMLElement | null;
      if (!soundElement) {
        return;
      }
      const soundId = Number(soundElement.getAttribute('data-sound-id'));
      if (!Number.isFinite(soundId)) {
        return;
      }
      event.preventDefault();
      event.stopPropagation();
      this.moveSound(soundId, direction);
    },
    moveSound(soundId: number, direction: string): void {
      const sourceId = ALL_GROUPS.find(groupId => this.columns[groupId].some(sound => sound.id === soundId));
      if (!sourceId) {
        return;
      }
      const source = this.columns[sourceId].slice();
      const currentIndex = source.findIndex(sound => sound.id === soundId);
      if (currentIndex < 0) {
        return;
      }
      const item = source.splice(currentIndex, 1)[0];

      if (direction === 'up' || direction === 'down') {
        const nextIndex = Math.max(0, Math.min(source.length, currentIndex + (direction === 'up' ? -1 : 1)));
        source.splice(nextIndex, 0, item);
        this.setColumn(sourceId, source);
      } else {
        let targetId: GroupId | undefined;
        if (sourceId === 'other') {
          targetId = direction === 'left' ? 'closing' : 'opening';
        } else {
          const index = MAIN_GROUPS.indexOf(sourceId);
          const nextIndex = index + (direction === 'left' ? -1 : 1);
          if (nextIndex >= 0 && nextIndex < MAIN_GROUPS.length) {
            targetId = MAIN_GROUPS[nextIndex];
          }
        }
        if (!targetId) {
          return;
        }
        const target = this.columns[targetId].slice();
        target.splice(Math.min(currentIndex, target.length), 0, item);
        this.setColumn(sourceId, source);
        this.setColumn(targetId, target);
      }

      this.scheduleSave();
      this.$nextTick(() => {
        const element = this.$el.querySelector('[data-sound-id="' + soundId + '"] .cf-sound__main, [data-sound-id="' + soundId + '"] .cf-sound__handle') as HTMLElement | null;
        if (element) {
          element.focus();
        }
      });
    },
  },
});
</script>

<style lang="scss">
.callflow {
  --cf-text: rgba(15, 23, 42, 0.92);
  --cf-muted: rgba(15, 23, 42, 0.6);
  --cf-col-bg: rgba(15, 23, 42, 0.035);
  --cf-col-border: rgba(15, 23, 42, 0.1);
  --cf-col-border-strong: rgba(15, 23, 42, 0.22);
  --cf-btn-bg: #fff;
  --cf-btn-border: rgba(15, 23, 42, 0.14);
  --cf-btn-border-strong: rgba(15, 23, 42, 0.28);
  --cf-btn-hover: #f3f5fa;
  --cf-btn-hover-strong: rgba(15, 23, 42, 0.08);
  --cf-opening: #0b7f93;
  --cf-opening-ink: #075f6e;
  --cf-opening-soft: rgba(11, 127, 147, 0.13);
  --cf-qualification: #6554d6;
  --cf-qualification-ink: #4a3bb8;
  --cf-qualification-soft: rgba(101, 84, 214, 0.13);
  --cf-closing: #b86e00;
  --cf-closing-ink: #8a5200;
  --cf-closing-soft: rgba(184, 110, 0, 0.14);
  --cf-yes: #1a8f58;
  --cf-no: #cc3338;
  display: flex;
  flex-direction: column;
  gap: 10px;
  height: calc(100vh - 283px);
  min-height: 180px;
  overflow: hidden;
  padding: 12px;
  box-sizing: border-box;
  color: var(--cf-text);
}

.theme--dark .callflow {
  --cf-text: rgba(255, 255, 255, 0.92);
  --cf-muted: rgba(255, 255, 255, 0.62);
  --cf-col-bg: rgba(255, 255, 255, 0.04);
  --cf-col-border: rgba(255, 255, 255, 0.09);
  --cf-col-border-strong: rgba(255, 255, 255, 0.2);
  --cf-btn-bg: rgba(255, 255, 255, 0.035);
  --cf-btn-border: rgba(255, 255, 255, 0.11);
  --cf-btn-border-strong: rgba(255, 255, 255, 0.22);
  --cf-btn-hover: rgba(255, 255, 255, 0.08);
  --cf-btn-hover-strong: rgba(255, 255, 255, 0.11);
}

.callflow__columns {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 12px;
  flex: 1;
  min-height: 0;
}

.callflow__tray {
  flex: 0 0 auto;
  max-height: 112px;
}

.cf-col {
  --cf-accent: var(--cf-opening);
  --cf-accent-ink: var(--cf-opening-ink);
  --cf-accent-soft: var(--cf-opening-soft);
  display: flex;
  flex-direction: column;
  min-height: 0;
  min-width: 0;
  background: var(--cf-col-bg);
  border: 1px solid var(--cf-col-border);
  border-top: 3px solid var(--cf-accent);
  border-radius: 10px;
}

.cf-col--qualification {
  --cf-accent: var(--cf-qualification);
  --cf-accent-ink: var(--cf-qualification-ink);
  --cf-accent-soft: var(--cf-qualification-soft);
}

.cf-col--closing {
  --cf-accent: var(--cf-closing);
  --cf-accent-ink: var(--cf-closing-ink);
  --cf-accent-soft: var(--cf-closing-soft);
}

.cf-col--other {
  --cf-accent: var(--cf-muted);
  --cf-accent-ink: var(--cf-text);
  --cf-accent-soft: var(--cf-btn-hover-strong);
}

.cf-col__head {
  display: flex;
  align-items: center;
  gap: 8px;
  min-height: 44px;
  padding: 8px 10px 6px 12px;
}

.cf-col__index {
  flex: none;
  display: grid;
  place-items: center;
  width: 22px;
  height: 22px;
  border-radius: 6px;
  font: 800 12px/1 ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
  color: var(--cf-accent-ink);
  background: var(--cf-accent-soft);
}

.cf-col__titlewrap {
  flex: 1;
  min-width: 0;
}

.cf-title {
  all: unset;
  box-sizing: border-box;
  max-width: 100%;
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 3px 6px;
  margin-left: -6px;
  border-radius: 6px;
  cursor: text;
  color: var(--cf-text);
}

.cf-title__text {
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  font-size: 13px;
  font-weight: 800;
  letter-spacing: 0.08em;
  text-transform: uppercase;
}

.cf-title__icon {
  color: var(--cf-muted);
  opacity: 0.65;
}

.cf-title:hover {
  background: var(--cf-btn-hover-strong);
}

.cf-edit {
  flex: 1;
  min-width: 0;
  display: flex;
  align-items: center;
  gap: 4px;
}

.cf-edit__input {
  flex: 1;
  min-width: 0;
  height: 30px;
  padding: 0 8px;
  border-radius: 6px;
  border: 1.5px solid var(--cf-accent);
  background: var(--cf-btn-bg);
  color: var(--cf-text);
  font-size: 13px;
  font-weight: 700;
  outline: none;
}

.cf-edit__btn {
  all: unset;
  box-sizing: border-box;
  display: grid;
  place-items: center;
  width: 28px;
  height: 28px;
  border-radius: 6px;
  cursor: pointer;
  color: var(--cf-muted);
}

.cf-edit__btn--ok {
  color: var(--cf-accent-ink);
  background: var(--cf-accent-soft);
}

.cf-col__count {
  flex: none;
  min-width: 22px;
  padding: 2px 7px;
  border-radius: 999px;
  text-align: center;
  font: 700 11px/1.4 ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
  color: var(--cf-muted);
  background: var(--cf-btn-hover-strong);
}

.cf-list {
  flex: 1;
  min-height: 64px;
  display: flex;
  flex-direction: column;
  gap: 6px;
  padding: 4px 10px 10px;
  overflow-y: auto;
  overflow-x: hidden;
  scrollbar-width: thin;
}

.cf-list:empty::before {
  content: "Drop recordings here";
  display: grid;
  place-items: center;
  min-height: 52px;
  border: 1.5px dashed var(--cf-col-border-strong);
  border-radius: 8px;
  font-size: 12px;
  color: var(--cf-muted);
  pointer-events: none;
}

.cf-col--tray .cf-list {
  flex-direction: row;
  flex-wrap: wrap;
  min-height: 46px;
  padding-bottom: 8px;
}

.cf-col--tray .cf-list > * {
  flex: 0 1 240px;
  min-width: 180px;
}

.cf-sound {
  --cf-stripe: var(--cf-accent);
  position: relative;
  display: flex;
  align-items: stretch;
  min-height: 44px;
  background: var(--cf-btn-bg);
  border: 1px solid var(--cf-btn-border);
  border-left: 3px solid var(--cf-stripe);
  border-radius: 8px;
  color: var(--cf-text);
  transition: background-color 0.12s ease, border-color 0.12s ease, transform 80ms ease;
}

.cf-sound--yes {
  --cf-stripe: var(--cf-yes);
}

.cf-sound--no {
  --cf-stripe: var(--cf-no);
}

.cf-sound:hover {
  background: var(--cf-btn-hover);
  border-color: var(--cf-btn-border-strong);
  border-left-color: var(--cf-stripe);
}

.cf-sound:active {
  transform: scale(0.99);
}

.cf-sound__handle {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  flex: none;
  color: var(--cf-muted);
  cursor: grab;
  touch-action: none;
  border-radius: 6px 0 0 6px;
  outline-offset: -2px;
}

.cf-sound__handle:hover,
.cf-sound__handle:focus-visible {
  color: var(--cf-text);
  background: var(--cf-btn-hover-strong);
}

.cf-sound__handle:active {
  cursor: grabbing;
}

.cf-icon {
  flex: none;
  display: block;
}

.cf-sound__main {
  all: unset;
  box-sizing: border-box;
  flex: 1;
  min-width: 0;
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 6px 6px 6px 2px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 600;
  line-height: 1.25;
}

.cf-sound__main:focus-visible {
  outline: 2px solid var(--cf-accent);
  outline-offset: 1px;
}

.cf-sound__code {
  flex: none;
  min-width: 28px;
  padding: 3px 6px;
  border-radius: 5px;
  text-align: center;
  font: 700 12px/1 ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
  font-variant-numeric: tabular-nums;
  color: var(--cf-accent-ink);
  background: var(--cf-accent-soft);
}

.cf-sound__label {
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.cf-sound--code-only .cf-sound__label {
  font: 700 15px/1.2 ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
  letter-spacing: 0.02em;
}

.cf-sound__meta {
  display: flex;
  align-items: center;
  gap: 6px;
  padding-right: 8px;
  color: var(--cf-muted);
}

.cf-sound__branch {
  padding: 2px 6px;
  border-radius: 4px;
  font-size: 10.5px;
  font-weight: 800;
  letter-spacing: 0.06em;
  color: var(--cf-stripe);
  border: 1px solid var(--cf-stripe);
}

.cf-sound__fav {
  color: #d6a83d;
}

.cf-ghost {
  background: var(--cf-accent-soft);
  border: 1.5px dashed var(--cf-accent);
  border-left-width: 1.5px;
  border-radius: 8px;
  min-height: 44px;
}

.cf-chosen {
  box-shadow: 0 6px 18px rgba(0, 0, 0, 0.28);
}

.cf-drag {
  opacity: 0.96 !important;
  box-shadow: 0 10px 28px rgba(0, 0, 0, 0.35);
  cursor: grabbing;
  z-index: 10000;
}

.callflow__foot {
  display: flex;
  align-items: center;
  gap: 14px;
  min-height: 25px;
  color: var(--cf-muted);
  font-size: 11px;
}

.callflow__status {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  white-space: nowrap;
}

.callflow__dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: #1a8f58;
}

.callflow__status--warn .callflow__dot {
  background: #cc3338;
}

.callflow__hint {
  flex: 1;
  text-align: center;
}

.callflow__reset {
  all: unset;
  cursor: pointer;
  color: var(--cf-muted);
  white-space: nowrap;
}

.callflow__reset:hover {
  color: var(--cf-text);
}

@media (max-width: 900px) {
  .callflow__columns {
    gap: 6px;
    grid-template-columns: repeat(3, minmax(180px, 1fr));
  }
  .callflow {
    overflow-x: auto;
  }
  .callflow__foot {
    min-width: 540px;
  }
}

@media (prefers-reduced-motion: reduce) {
  .cf-sound,
  .cf-sound--fired::after {
    transition: none;
    animation: none;
  }
}
</style>
