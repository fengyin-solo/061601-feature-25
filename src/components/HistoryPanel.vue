<script setup lang="ts">
import { ref, computed } from 'vue'
import { useGameStore } from '../stores/gameStore'
import type { RollbackDiff } from '../stores/gameStore'
import { getTimeLabel } from '../utils/gameUtils'
import gameConfig from '../config/gameConfig'

const emit = defineEmits<{
  (e: 'close'): void
}>()

const gameStore = useGameStore()

const history = computed(() => [...gameStore.history].reverse())
const selectedIndex = ref<number | null>(null)
const showPreview = ref(false)
const rollbackDiff = ref<RollbackDiff | null>(null)
const confirmed = ref(false)
const hoveredIndex = ref<number | null>(null)

function openPreview(index: number) {
  const realIndex = gameStore.history.length - 1 - index
  selectedIndex.value = realIndex
  rollbackDiff.value = gameStore.getRollbackDiff(realIndex)
  showPreview.value = true
  confirmed.value = false
}

function closePreview() {
  showPreview.value = false
  selectedIndex.value = null
  rollbackDiff.value = null
  confirmed.value = false
}

function toggleConfirm() {
  confirmed.value = !confirmed.value
}

function doRollback() {
  if (selectedIndex.value === null) return
  gameStore.rollbackToStep(selectedIndex.value)
  closePreview()
  emit('close')
}

function getStepSummary(snapshot: any): string {
  const lastLog = snapshot.logs[snapshot.logs.length - 1]
  if (lastLog) {
    return lastLog.message.length > 30 ? lastLog.message.substring(0, 30) + '...' : lastLog.message
  }
  return '历史步骤'
}

function getRarityLabel(rarity: string): string {
  const map: Record<string, string> = {
    common: '普通',
    rare: '稀有',
    epic: '史诗',
    legendary: '传说'
  }
  return map[rarity] || rarity
}

function getRarityClass(rarity: string): string {
  return `rarity-${rarity}`
}

const criticalWarnings = computed(() => {
  if (!rollbackDiff.value) return []
  const warnings: { type: string; icon: string; text: string; level: 'danger' | 'warning' | 'info' }[] = []
  const d = rollbackDiff.value
  if (d.lostCharacters.length > 0) {
    warnings.push({
      type: 'character',
      icon: '🚨',
      text: `将失去已解锁角色：${d.lostCharacters.map(c => c.name).join('、')}`,
      level: 'danger'
    })
  }
  if (d.lostCards.length > 0) {
    const legendaryCount = d.lostCards.filter(c => c.rarity === 'legendary').length
    const epicCount = d.lostCards.filter(c => c.rarity === 'epic').length
    const rareCount = d.lostCards.filter(c => c.rarity === 'rare').length
    let text = `将失去 ${d.lostCards.length} 张卡牌`
    const details: string[] = []
    if (legendaryCount > 0) details.push(`${legendaryCount}张传说`)
    if (epicCount > 0) details.push(`${epicCount}张史诗`)
    if (rareCount > 0) details.push(`${rareCount}张稀有`)
    if (details.length > 0) text += `（${details.join('、')}）`
    warnings.push({
      type: 'card',
      icon: legendaryCount > 0 || epicCount > 0 ? '🚨' : '⚠️',
      text,
      level: legendaryCount > 0 || epicCount > 0 ? 'danger' : 'warning'
    })
  }
  if (d.lostEvents.length > 0) {
    warnings.push({
      type: 'event',
      icon: '📖',
      text: `已触发的 ${d.lostEvents.length} 个事件会重新变为可触发状态`,
      level: 'info'
    })
  }
  const bigAffinityDrops = d.characterDiffs.filter(c => c.affinityChange <= -10)
  if (bigAffinityDrops.length > 0) {
    warnings.push({
      type: 'affinity',
      icon: '💔',
      text: `${bigAffinityDrops.map(c => `${c.name}(${c.affinityChange})`).join('、')} 好感度大幅下降`,
      level: 'warning'
    })
  }
  if (d.daysLost >= 3) {
    warnings.push({
      type: 'time',
      icon: '⏳',
      text: `将回退 ${d.daysLost} 天的游戏进度`,
      level: 'warning'
    })
  }
  return warnings
})
</script>

<template>
  <Teleport to="body">
    <div class="modal-overlay" @click.self="emit('close')">
      <div class="modal-content history-modal">
        <div class="modal-header">
          <h2>📜 历史记录</h2>
          <button class="close-btn" @click="emit('close')">✕</button>
        </div>

        <div class="history-hint">
          💡 点击任意历史步骤可查看回退预览
        </div>

        <div class="history-list">
          <div
            v-for="(snapshot, index) in history"
            :key="index"
            class="history-item"
            :class="{ 'is-current': index === 0, 'is-hovered': hoveredIndex === index }"
            @click="openPreview(index)"
            @mouseenter="hoveredIndex = index"
            @mouseleave="hoveredIndex = null"
          >
            <div class="step-badge">
              <span v-if="index === 0">当前</span>
              <span v-else>{{ history.length - index }}</span>
            </div>
            <div class="step-info">
              <span class="step-time">
                第{{ snapshot.day }}天 {{ getTimeLabel(snapshot.timeSlot) }}
              </span>
              <span class="step-summary">{{ getStepSummary(snapshot) }}</span>
            </div>
            <div class="step-meta">
              <span class="step-resources">💰 {{ snapshot.resources }}</span>
              <span class="step-actions">⚡ {{ snapshot.actionsRemaining }}</span>
            </div>
            <div v-if="index !== 0" class="step-preview-icon">👁️</div>
          </div>

          <div v-if="history.length === 0" class="empty-history">
            暂无历史记录
          </div>
        </div>
      </div>
    </div>

    <Teleport to="body">
      <div v-if="showPreview && rollbackDiff" class="modal-overlay preview-overlay" @click.self="closePreview">
        <div class="modal-content preview-modal">
          <div class="modal-header">
            <h2>⏪ 回退预览</h2>
            <button class="close-btn" @click="closePreview">✕</button>
          </div>

          <div class="preview-summary">
            <div class="summary-banner">
              回退到
              <span class="target-time">
                第 {{ rollbackDiff.targetSnapshot.day }} 天
                {{ getTimeLabel(rollbackDiff.targetSnapshot.timeSlot) }}
              </span>
              ，丢失
              <span class="lost-steps">{{ rollbackDiff.stepsLost }} 步</span>
              <span v-if="rollbackDiff.daysLost > 0">
                （{{ rollbackDiff.daysLost }} 天）
              </span>
              进度
            </div>
          </div>

          <div v-if="criticalWarnings.length > 0" class="warnings-section">
            <div class="section-title">
              <span>⚠️ 关键提醒</span>
            </div>
            <div class="warnings-list">
              <div
                v-for="(warning, idx) in criticalWarnings"
                :key="idx"
                class="warning-item"
                :class="`level-${warning.level}`"
              >
                <span class="warning-icon">{{ warning.icon }}</span>
                <span class="warning-text">{{ warning.text }}</span>
              </div>
            </div>
          </div>

          <div class="diff-sections">
            <div class="diff-section">
              <div class="section-title">
                <span>💎 卡牌变化</span>
                <span class="section-count">{{ rollbackDiff.lostCards.length }} 张丢失</span>
              </div>
              <div v-if="rollbackDiff.lostCards.length > 0" class="cards-grid">
                <div
                  v-for="card in rollbackDiff.lostCards"
                  :key="card.id"
                  class="card-item"
                  :class="getRarityClass(card.rarity)"
                >
                  <div class="card-icon">{{ card.image }}</div>
                  <div class="card-name">{{ card.name }}</div>
                  <div class="card-rarity">{{ getRarityLabel(card.rarity) }}</div>
                </div>
              </div>
              <div v-else class="empty-diff">无卡牌丢失</div>
            </div>

            <div class="diff-section">
              <div class="section-title">
                <span>👥 角色状态变化</span>
              </div>
              <div class="characters-diff">
                <div
                  v-for="diff in rollbackDiff.characterDiffs"
                  :key="diff.id"
                  class="char-diff-item"
                >
                  <div class="char-avatar">{{ diff.avatar }}</div>
                  <div class="char-info">
                    <div class="char-name">
                      {{ diff.name }}
                      <span v-if="diff.unlockedChange === 'locked'" class="tag tag-danger">将锁定</span>
                    </div>
                    <div class="char-stats">
                      <span class="stat affinity" :class="{ negative: diff.affinityChange < 0, positive: diff.affinityChange > 0 }">
                        💕 好感 {{ diff.affinityChange > 0 ? '+' : '' }}{{ diff.affinityChange }}
                      </span>
                      <span class="stat mood" :class="{ negative: diff.moodChange < 0, positive: diff.moodChange > 0 }">
                        😊 心情 {{ diff.moodChange > 0 ? '+' : '' }}{{ diff.moodChange }}
                      </span>
                    </div>
                  </div>
                </div>
              </div>
            </div>

            <div class="diff-section">
              <div class="section-title">
                <span>📊 资源与进度</span>
              </div>
              <div class="resource-compare">
                <div class="compare-item">
                  <span class="compare-label">代币</span>
                  <div class="compare-values">
                    <span class="current-value">
                      {{ rollbackDiff.currentSnapshot.resources }}
                    </span>
                    <span class="arrow">→</span>
                    <span class="target-value">
                      {{ rollbackDiff.targetSnapshot.resources }}
                    </span>
                    <span
                      class="change-tag"
                      :class="{ negative: rollbackDiff.resourcesChange < 0, positive: rollbackDiff.resourcesChange > 0 }"
                    >
                      {{ rollbackDiff.resourcesChange > 0 ? '+' : '' }}{{ rollbackDiff.resourcesChange }}
                    </span>
                  </div>
                </div>
                <div class="compare-item">
                  <span class="compare-label">行动力</span>
                  <div class="compare-values">
                    <span class="current-value">
                      {{ rollbackDiff.currentSnapshot.actionsRemaining }}
                    </span>
                    <span class="arrow">→</span>
                    <span class="target-value">
                      {{ rollbackDiff.targetSnapshot.actionsRemaining }}
                    </span>
                  </div>
                </div>
                <div class="compare-item">
                  <span class="compare-label">已触发事件</span>
                  <div class="compare-values">
                    <span class="current-value">
                      {{ rollbackDiff.currentSnapshot.triggeredEvents.length }}
                    </span>
                    <span class="arrow">→</span>
                    <span class="target-value">
                      {{ rollbackDiff.targetSnapshot.triggeredEvents.length }}
                    </span>
                  </div>
                </div>
              </div>
              <div v-if="rollbackDiff.lostEvents.length > 0" class="lost-events">
                <div class="lost-events-title">受影响的事件：</div>
                <div class="lost-events-list">
                  <span v-for="ev in rollbackDiff.lostEvents" :key="ev.id" class="event-tag">
                    📖 {{ ev.title }}
                  </span>
                </div>
              </div>
            </div>
          </div>

          <div class="preview-actions">
            <label class="confirm-checkbox">
              <input type="checkbox" v-model="confirmed" @change="toggleConfirm" />
              <span>我已了解以上影响，确认要回退</span>
            </label>
            <div class="action-buttons">
              <button class="btn btn-secondary" @click="closePreview">取消</button>
              <button
                class="btn btn-danger"
                :disabled="!confirmed"
                @click="doRollback"
              >
                ⏪ 确认回退
              </button>
            </div>
          </div>
        </div>
      </div>
    </Teleport>
  </Teleport>
</template>

<style scoped>
.history-modal {
  padding: 24px;
  max-width: 500px;
}

.preview-overlay {
  z-index: 2000;
}

.preview-modal {
  padding: 24px;
  max-width: 680px;
  max-height: 85vh;
  display: flex;
  flex-direction: column;
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
  flex-shrink: 0;
}

.modal-header h2 {
  font-size: 20px;
  font-weight: 600;
}

.close-btn {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  background: var(--bg-tertiary);
  font-size: 16px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  border: none;
  transition: all 0.2s;
}

.close-btn:hover {
  background: var(--accent-light);
}

.history-hint {
  padding: 10px 14px;
  background: #dbeafe;
  color: #1e40af;
  border-radius: var(--radius-sm);
  font-size: 13px;
  margin-bottom: 16px;
}

[data-theme='dark'] .history-hint {
  background: #1e3a5f;
  color: #93c5fd;
}

.history-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
  max-height: 400px;
  overflow-y: auto;
  padding-right: 8px;
}

.history-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 14px;
  background: var(--bg-tertiary);
  border-radius: var(--radius-md);
  cursor: pointer;
  transition: all 0.2s;
  border: 2px solid transparent;
}

.history-item:hover,
.history-item.is-hovered {
  background: var(--accent-light);
  transform: translateX(4px);
  border-color: var(--accent-primary);
}

.history-item.is-current {
  background: linear-gradient(135deg, var(--accent-light), #e0f2fe);
  border-color: var(--accent-primary);
}

[data-theme='dark'] .history-item.is-current {
  background: linear-gradient(135deg, #1e3a5f, #1e40af33);
}

.step-badge {
  width: 48px;
  height: 32px;
  border-radius: 16px;
  background: var(--accent-primary);
  color: white;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 600;
  font-size: 12px;
  flex-shrink: 0;
}

.history-item.is-current .step-badge {
  background: linear-gradient(135deg, #10b981, #059669);
  width: auto;
  padding: 0 12px;
}

.step-info {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.step-time {
  font-weight: 500;
  font-size: 14px;
}

.step-summary {
  font-size: 12px;
  color: var(--text-secondary);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.step-meta {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 2px;
  font-size: 12px;
  color: var(--text-muted);
}

.step-preview-icon {
  opacity: 0;
  transition: opacity 0.2s;
  font-size: 18px;
}

.history-item:hover .step-preview-icon {
  opacity: 1;
}

.empty-history {
  text-align: center;
  padding: 40px;
  color: var(--text-muted);
  font-size: 14px;
}

.preview-summary {
  margin-bottom: 16px;
  flex-shrink: 0;
}

.summary-banner {
  padding: 14px 16px;
  background: linear-gradient(135deg, #fef3c7, #fde68a);
  border-radius: var(--radius-md);
  font-size: 14px;
  text-align: center;
  color: #92400e;
  line-height: 1.6;
}

[data-theme='dark'] .summary-banner {
  background: linear-gradient(135deg, #78350f66, #451a0366);
  color: #fcd34d;
}

.target-time {
  font-weight: 700;
  color: #b45309;
}

[data-theme='dark'] .target-time {
  color: #fbbf24;
}

.lost-steps {
  font-weight: 700;
  color: #dc2626;
}

.warnings-section {
  margin-bottom: 16px;
  flex-shrink: 0;
}

.section-title {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 8px 0;
  font-weight: 600;
  font-size: 14px;
  color: var(--text-primary);
  border-bottom: 1px solid var(--border-color, #e5e7eb);
  margin-bottom: 10px;
}

.section-count {
  font-size: 12px;
  font-weight: 500;
  color: var(--text-secondary);
  background: var(--bg-tertiary);
  padding: 2px 10px;
  border-radius: 12px;
}

.warnings-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.warning-item {
  display: flex;
  align-items: flex-start;
  gap: 10px;
  padding: 10px 12px;
  border-radius: var(--radius-sm);
  font-size: 13px;
  line-height: 1.5;
}

.warning-item.level-danger {
  background: #fef2f2;
  color: #991b1b;
  border-left: 4px solid #ef4444;
}

[data-theme='dark'] .warning-item.level-danger {
  background: #7f1d1d33;
  color: #fca5a5;
}

.warning-item.level-warning {
  background: #fffbeb;
  color: #92400e;
  border-left: 4px solid #f59e0b;
}

[data-theme='dark'] .warning-item.level-warning {
  background: #78350f33;
  color: #fcd34d;
}

.warning-item.level-info {
  background: #eff6ff;
  color: #1e40af;
  border-left: 4px solid #3b82f6;
}

[data-theme='dark'] .warning-item.level-info {
  background: #1e3a8a33;
  color: #93c5fd;
}

.warning-icon {
  font-size: 16px;
  flex-shrink: 0;
}

.diff-sections {
  flex: 1;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 16px;
  padding-right: 4px;
}

.cards-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(100px, 1fr));
  gap: 8px;
}

.card-item {
  padding: 10px;
  border-radius: var(--radius-sm);
  text-align: center;
  background: var(--bg-tertiary);
  border: 2px solid transparent;
  transition: transform 0.2s;
}

.card-item:hover {
  transform: translateY(-2px);
}

.card-item.rarity-common {
  border-color: #9ca3af;
  background: #f9fafb;
}

.card-item.rarity-rare {
  border-color: #3b82f6;
  background: #eff6ff;
}

.card-item.rarity-epic {
  border-color: #8b5cf6;
  background: #f5f3ff;
}

.card-item.rarity-legendary {
  border-color: #f59e0b;
  background: linear-gradient(135deg, #fffbeb, #fef3c7);
  box-shadow: 0 0 12px rgba(245, 158, 11, 0.3);
}

[data-theme='dark'] .card-item.rarity-common { background: #1f2937; }
[data-theme='dark'] .card-item.rarity-rare { background: #1e3a8a33; }
[data-theme='dark'] .card-item.rarity-epic { background: #581c8733; }
[data-theme='dark'] .card-item.rarity-legendary { background: linear-gradient(135deg, #78350f33, #451a0333); }

.card-icon {
  font-size: 28px;
  margin-bottom: 4px;
}

.card-name {
  font-size: 12px;
  font-weight: 600;
  margin-bottom: 2px;
}

.card-rarity {
  font-size: 10px;
  color: var(--text-secondary);
}

.empty-diff {
  padding: 16px;
  text-align: center;
  color: var(--text-muted);
  font-size: 13px;
  background: var(--bg-tertiary);
  border-radius: var(--radius-sm);
}

.characters-diff {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.char-diff-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px 12px;
  background: var(--bg-tertiary);
  border-radius: var(--radius-sm);
}

.char-avatar {
  font-size: 28px;
  width: 44px;
  height: 44px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--bg-secondary, #f3f4f6);
  border-radius: 50%;
  flex-shrink: 0;
}

.char-info {
  flex: 1;
  min-width: 0;
}

.char-name {
  font-weight: 600;
  font-size: 14px;
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 4px;
}

.tag {
  font-size: 10px;
  padding: 2px 8px;
  border-radius: 10px;
  font-weight: 500;
}

.tag-danger {
  background: #fee2e2;
  color: #991b1b;
}

[data-theme='dark'] .tag-danger {
  background: #7f1d1d66;
  color: #fca5a5;
}

.char-stats {
  display: flex;
  gap: 12px;
  font-size: 12px;
  flex-wrap: wrap;
}

.stat.negative { color: #dc2626; }
.stat.positive { color: #16a34a; }

.resource-compare {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-bottom: 12px;
}

.compare-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 12px;
  background: var(--bg-tertiary);
  border-radius: var(--radius-sm);
}

.compare-label {
  font-size: 13px;
  font-weight: 500;
  color: var(--text-secondary);
}

.compare-values {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 13px;
}

.current-value {
  color: var(--text-muted);
  text-decoration: line-through;
}

.arrow {
  color: var(--text-muted);
}

.target-value {
  font-weight: 600;
  color: var(--accent-primary);
}

.change-tag {
  font-size: 11px;
  padding: 2px 8px;
  border-radius: 10px;
  font-weight: 600;
}

.change-tag.negative {
  background: #fee2e2;
  color: #dc2626;
}

.change-tag.positive {
  background: #dcfce7;
  color: #16a34a;
}

[data-theme='dark'] .change-tag.negative { background: #7f1d1d66; }
[data-theme='dark'] .change-tag.positive { background: #14532d66; color: #4ade80; }

.lost-events-title {
  font-size: 12px;
  color: var(--text-secondary);
  margin-bottom: 6px;
}

.lost-events-list {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.event-tag {
  font-size: 12px;
  padding: 4px 10px;
  background: #eff6ff;
  color: #1e40af;
  border-radius: 12px;
}

[data-theme='dark'] .event-tag {
  background: #1e3a8a33;
  color: #93c5fd;
}

.preview-actions {
  margin-top: 16px;
  padding-top: 16px;
  border-top: 1px solid var(--border-color, #e5e7eb);
  display: flex;
  flex-direction: column;
  gap: 12px;
  flex-shrink: 0;
}

.confirm-checkbox {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 13px;
  color: var(--text-secondary);
  cursor: pointer;
}

.confirm-checkbox input[type="checkbox"] {
  width: 16px;
  height: 16px;
  cursor: pointer;
}

.action-buttons {
  display: flex;
  gap: 10px;
  justify-content: flex-end;
}

.btn {
  padding: 10px 20px;
  border-radius: var(--radius-md);
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  border: none;
  transition: all 0.2s;
}

.btn-secondary {
  background: var(--bg-tertiary);
  color: var(--text-primary);
}

.btn-secondary:hover {
  background: var(--bg-secondary, #e5e7eb);
}

.btn-danger {
  background: linear-gradient(135deg, #ef4444, #dc2626);
  color: white;
}

.btn-danger:hover:not(:disabled) {
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(239, 68, 68, 0.4);
}

.btn-danger:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
</style>
