<script setup lang="ts">
import { computed } from 'vue'
import { useGameStore } from '../stores/gameStore'
import { getTimeLabel, getTimeIcon } from '../utils/gameUtils'

const emit = defineEmits<{
  (e: 'toggle-save'): void
  (e: 'toggle-cards'): void
  (e: 'toggle-history'): void
  (e: 'toggle-theme'): void
  (e: 'reset'): void
}>()

const gameStore = useGameStore()

const rollbackLabel = computed(() => {
  if (!gameStore.rollbackOrigin) return ''
  const o = gameStore.rollbackOrigin
  return `从第 ${o.day} 天 ${getTimeLabel(o.timeSlot)} 回退`
})
</script>

<template>
  <header class="top-bar card">
    <div class="game-title">
      <span class="title-icon">💝</span>
      <h1>恋爱物语</h1>
      <span v-if="gameStore.rolledBack" class="rollback-badge" :title="rollbackLabel">
        <span class="rollback-icon">⏪</span>
        <span class="rollback-text">已回退</span>
        <span v-if="gameStore.rollbackOrigin" class="rollback-detail"> · {{ rollbackLabel }}</span>
      </span>
    </div>

    <div class="status-info">
      <div class="status-item day">
        <span class="status-icon">📅</span>
        <span>第 {{ gameStore.day }} 天</span>
      </div>
      <div class="status-item time">
        <span class="status-icon">{{ getTimeIcon(gameStore.timeSlot) }}</span>
        <span>{{ getTimeLabel(gameStore.timeSlot) }}</span>
      </div>
      <div class="status-item actions">
        <span class="status-icon">⚡</span>
        <span>行动力 {{ gameStore.actionsRemaining }}</span>
      </div>
      <div class="status-item resources">
        <span class="status-icon">💰</span>
        <span>{{ gameStore.resources }} 代币</span>
      </div>
    </div>

    <div class="toolbar">
      <button class="toolbar-btn" @click="emit('toggle-cards')" title="卡牌收藏">
        🎴
      </button>
      <button
        class="toolbar-btn history-btn"
        :class="{ 'has-rollback': gameStore.rolledBack }"
        @click="emit('toggle-history')"
        title="历史记录"
      >
        📜
        <span v-if="gameStore.rolledBack" class="btn-dot"></span>
      </button>
      <button class="toolbar-btn" @click="emit('toggle-save')" title="存档/读档">
        💾
      </button>
      <button class="toolbar-btn" @click="emit('toggle-theme')" title="切换主题">
        {{ gameStore.darkMode ? '☀️' : '🌙' }}
      </button>
      <button class="toolbar-btn reset" @click="emit('reset')" title="重新开始">
        🔄
      </button>
    </div>
  </header>
</template>

<style scoped>
.top-bar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 24px;
  gap: 20px;
}

.game-title {
  display: flex;
  align-items: center;
  gap: 10px;
  flex-wrap: wrap;
}

.title-icon {
  font-size: 28px;
}

.game-title h1 {
  font-size: 20px;
  font-weight: 700;
  background: linear-gradient(135deg, var(--accent-primary), var(--accent-secondary));
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.rollback-badge {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 4px 12px;
  background: linear-gradient(135deg, #fef3c7, #fde68a);
  color: #92400e;
  border-radius: 16px;
  font-size: 12px;
  font-weight: 600;
  border: 1px solid #f59e0b;
  animation: rollbackPulse 2s ease-in-out infinite;
  margin-left: 4px;
  max-width: 320px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

[data-theme='dark'] .rollback-badge {
  background: linear-gradient(135deg, #78350f66, #451a0366);
  color: #fcd34d;
  border-color: #d97706;
}

.rollback-icon {
  font-size: 14px;
}

.rollback-text {
  font-weight: 700;
}

.rollback-detail {
  font-weight: 500;
  opacity: 0.85;
}

@keyframes rollbackPulse {
  0%, 100% {
    box-shadow: 0 0 0 0 rgba(245, 158, 11, 0.4);
  }
  50% {
    box-shadow: 0 0 0 6px rgba(245, 158, 11, 0);
  }
}

.status-info {
  display: flex;
  gap: 20px;
  flex: 1;
  justify-content: center;
}

.status-item {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 8px 16px;
  background: var(--bg-tertiary);
  border-radius: var(--radius-md);
  font-size: 14px;
  font-weight: 500;
}

.status-icon {
  font-size: 18px;
}

.toolbar {
  display: flex;
  gap: 8px;
}

.toolbar-btn {
  position: relative;
  width: 40px;
  height: 40px;
  border-radius: var(--radius-md);
  background: var(--bg-tertiary);
  font-size: 18px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  border: none;
  transition: all 0.2s;
}

.toolbar-btn:hover {
  background: var(--accent-light);
  transform: scale(1.05);
}

.toolbar-btn.has-rollback {
  background: linear-gradient(135deg, #fef3c7, #fde68a);
  border: 2px solid #f59e0b;
}

[data-theme='dark'] .toolbar-btn.has-rollback {
  background: linear-gradient(135deg, #78350f66, #451a0366);
  border-color: #d97706;
}

.btn-dot {
  position: absolute;
  top: 4px;
  right: 4px;
  width: 8px;
  height: 8px;
  background: #ef4444;
  border-radius: 50%;
  border: 1px solid white;
  animation: dotPulse 1.5s ease-in-out infinite;
}

@keyframes dotPulse {
  0%, 100% { transform: scale(1); opacity: 1; }
  50% { transform: scale(1.3); opacity: 0.7; }
}

.toolbar-btn.reset:hover {
  background: #fee2e2;
}

@media (max-width: 768px) {
  .top-bar {
    flex-wrap: wrap;
  }
  
  .status-info {
    order: 3;
    width: 100%;
    justify-content: space-around;
    gap: 8px;
  }
  
  .status-item {
    padding: 6px 10px;
    font-size: 12px;
  }
  
  .game-title h1 {
    font-size: 18px;
  }

  .rollback-badge {
    font-size: 11px;
    padding: 3px 10px;
    max-width: 220px;
  }

  .rollback-detail {
    display: none;
  }
}
</style>
