<script setup lang="ts">
import { computed, ref, onUnmounted } from 'vue'
import { NPopconfirm, NCheckbox, NTooltip } from 'naive-ui'
import { useI18n } from 'vue-i18n'
import type { Session } from '@/stores/hermes/chat'
import { useAppStore } from '@/stores/hermes/app'
import { useProfilesStore } from '@/stores/hermes/profiles'
import ProfileAvatar from '@/components/hermes/profiles/ProfileAvatar.vue'
import { formatTimestampMs } from '@/shared/session-display'

const props = withDefaults(defineProps<{
  session: Session
  active: boolean
  pinned: boolean
  canDelete: boolean
  streaming?: boolean
  selectable?: boolean
  selected?: boolean
  showProfile?: boolean
  to?: string
}>(), {
  showProfile: true,
})

const emit = defineEmits<{
  select: []
  contextmenu: [event: MouseEvent]
  delete: []
  'toggle-select': []
}>()

const { t } = useI18n()
const appStore = useAppStore()
const profilesStore = useProfilesStore()
const profileName = computed(() => props.session.profile || 'default')
const profileAvatar = computed(() => profilesStore.profiles.find(profile => profile.name === profileName.value)?.avatar)
const profileHasModels = computed(() => {
  const profileModels = appStore.profileModelGroups.find(profile => profile.profile === profileName.value)
  return !!profileModels?.groups?.some(group => group.models.length > 0)
})
const profileModelsMissing = computed(() =>
  appStore.profileModelGroups.length > 0 && !profileHasModels.value,
)
const sessionAgentLogo = computed(() => {
  if (props.session.source === 'coding_agent') {
    if (props.session.codingAgentId === 'codex' || props.session.agent === 'codex') {
      return { label: 'Codex', src: '/coding-agents/codex-openai.png' }
    }
    return { label: 'Claude Code', src: '/coding-agents/claude-code.svg' }
  }
  return { label: 'Hermes', src: '/coding-agents/hermes.png' }
})

let longPressTimer: ReturnType<typeof setTimeout> | null = null
const longPressTriggered = ref(false)

function onTouchStart(e: TouchEvent) {
  longPressTriggered.value = false
  longPressTimer = setTimeout(() => {
    longPressTriggered.value = true
    const touch = e.touches[0]
    const syntheticEvent = new MouseEvent('contextmenu', {
      clientX: touch.clientX,
      clientY: touch.clientY,
      bubbles: true,
    })
    emit('contextmenu', syntheticEvent)
  }, 500)
}

function onTouchEnd() {
  if (longPressTimer) {
    clearTimeout(longPressTimer)
    longPressTimer = null
  }
}

function onTouchMove() {
  if (longPressTimer) {
    clearTimeout(longPressTimer)
    longPressTimer = null
  }
}

function isModifiedNavigation(event?: MouseEvent) {
  return !!event && (event.metaKey || event.ctrlKey || event.shiftKey || event.altKey || event.button !== 0)
}

function onClick(event?: MouseEvent) {
  if (longPressTriggered.value) {
    longPressTriggered.value = false
    event?.preventDefault()
    return
  }
  if (isModifiedNavigation(event)) return
  if (props.to && !props.selectable) event?.preventDefault()
  emit('select')
}

onUnmounted(() => {
  if (longPressTimer) clearTimeout(longPressTimer)
})
</script>

<template>
  <component
    :is="selectable || !to ? 'button' : 'a'"
    class="session-item"
    :class="{ active, 'batch-mode': selectable, 'missing-models': profileModelsMissing }"
    :aria-current="active ? 'page' : undefined"
    :href="!selectable ? to : undefined"
    :type="selectable || !to ? 'button' : undefined"
    @click="onClick"
    @contextmenu="emit('contextmenu', $event)"
    @touchstart="onTouchStart"
    @touchend="onTouchEnd"
    @touchmove="onTouchMove"
  >
    <div v-if="selectable" class="session-item-checkbox">
      <NCheckbox :checked="selected" @click.stop="emit('toggle-select')" />
    </div>
    <div class="session-item-content">
      <span class="session-item-title-row">
        <span class="session-item-title-main">
          <span v-if="pinned" class="session-item-pin" aria-hidden="true">
            <svg width="11" height="11" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
              <path d="M12 17v5" />
              <path d="M5 8l14 0" />
              <path d="M8 3l8 0 0 5 3 5-14 0 3-5z" />
            </svg>
          </span>
          <span class="session-item-title">
            <svg v-if="streaming" class="session-item-streaming" width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round"><path d="M12 2v4M12 18v4M4.93 4.93l2.83 2.83M16.24 16.24l2.83 2.83M2 12h4M18 12h4M4.93 19.07l2.83-2.83M16.24 7.76l2.83-2.83"/></svg>
            {{ session.title }}
          </span>
          <NTooltip v-if="profileModelsMissing" trigger="click" placement="top">
            <template #trigger>
              <button class="session-item-warning" type="button" @click.stop.prevent>
                !
              </button>
            </template>
            {{ t('chat.profileMissingModelsTip', { profile: profileName }) }}
          </NTooltip>
        </span>
        <span class="session-item-time">{{ formatTimestampMs(session.createdAt) }}</span>
      </span>
      <span class="session-item-agent-row">
        <img
          class="session-item-agent-logo"
          :src="sessionAgentLogo.src"
          :alt="sessionAgentLogo.label"
        >
        <span v-if="props.showProfile" class="session-item-profile">
          <ProfileAvatar class="session-item-profile-avatar" :name="profileName" :avatar="profileAvatar" :size="16" />
          <span class="session-item-profile-name">{{ profileName }}</span>
        </span>
      </span>
    </div>
    <NPopconfirm v-if="canDelete && !selectable" @positive-click="emit('delete')">
      <template #trigger>
        <button class="session-item-delete" @click.stop.prevent>
          <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><line x1="18" y1="6" x2="6" y2="18"/><line x1="6" y1="6" x2="18" y2="18"/></svg>
        </button>
      </template>
      {{ t('chat.deleteSession') }}
    </NPopconfirm>
  </component>
</template>

<style scoped>
.session-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
  padding: 8px 10px;
  border: none;
  background: none;
  border-radius: var(--radius-sm);
  cursor: pointer;
  text-align: left;
  text-decoration: none;
  color: var(--text-secondary);
  transition: all var(--transition-fast);
  margin-bottom: 2px;
}

.session-item:hover {
  background: rgba(var(--accent-primary-rgb), 0.06);
  color: var(--text-primary);
}

.session-item:hover .session-item-delete {
  opacity: 1;
  pointer-events: auto;
}

.session-item:focus-within .session-item-delete {
  opacity: 1;
  pointer-events: auto;
}

.session-item.active {
  background: rgba(var(--accent-primary-rgb), 0.12);
  color: var(--text-primary);
  font-weight: 500;
  border-radius: 6px;
}

.session-item.active .session-item-title {
  color: var(--accent-primary);
}

.session-item.missing-models {
  color: #b42318;
  background: rgba(220, 38, 38, 0.08);
}

.session-item.missing-models .session-item-title,
.session-item.missing-models .session-item-profile-name,
.session-item.missing-models .session-item-time {
  color: #b42318;
}

.session-item.missing-models:hover {
  background: rgba(220, 38, 38, 0.12);
}

.session-item-content {
  flex: 1;
  overflow: hidden;
}

.session-item-title-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  min-width: 0;
}

.session-item-title-main {
  display: flex;
  align-items: center;
  gap: 6px;
  min-width: 0;
}

.session-item-title {
  display: block;
  flex: 1 1 auto;
  min-width: 0;
  font-size: 13px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.session-item-streaming {
  display: inline-block;
  flex-shrink: 0;
  margin-right: 4px;
  vertical-align: middle;
  animation: spin 1.2s linear infinite;
  color: var(--accent-primary);
}

@keyframes spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

.session-item-pin {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  color: var(--accent-primary);
}

.session-item-time {
  flex: 0 0 auto;
  font-size: 11px;
  color: var(--text-muted);
}

.session-item-delete {
  flex-shrink: 0;
  opacity: 0;
  pointer-events: none;
  padding: 2px;
  border: none;
  background: none;
  color: var(--text-muted);
  cursor: pointer;
  border-radius: 3px;
  transition: all var(--transition-fast);
}

.session-item-delete:hover {
  color: var(--error);
  background: rgba(var(--error-rgb), 0.1);
}

@media (hover: none) {
  .session-item-delete {
    opacity: 0.5;
    pointer-events: auto;
  }
}

.session-item-profile {
  display: flex;
  align-items: center;
  gap: 5px;
  min-width: 0;
}

.session-item-profile-avatar {
  background: var(--bg-secondary);
}

.session-item-profile-name {
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  font-size: 11px;
  line-height: 16px;
  color: var(--text-muted);
}

.session-item-warning {
  flex-shrink: 0;
  width: 16px;
  height: 16px;
  border: 1px solid rgba(180, 35, 24, 0.35);
  border-radius: 50%;
  background: rgba(220, 38, 38, 0.1);
  color: #b42318;
  font-size: 11px;
  font-weight: 700;
  line-height: 14px;
  cursor: pointer;
}

.session-item-agent-row {
  display: flex;
  align-items: center;
  gap: 4px;
  min-width: 0;
  margin-top: 4px;
}

.session-item-agent-logo {
  flex: 0 0 auto;
  width: 18px;
  height: 18px;
  padding: 2px;
  border-radius: 50%;
  object-fit: contain;
  background: #fff;
}
</style>
