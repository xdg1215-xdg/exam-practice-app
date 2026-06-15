<script setup>
import { computed, ref, onMounted, watch, onUnmounted } from 'vue'

const props = defineProps({
  question: { type: Object, required: true },
  index: { type: Number, required: true },
  total: { type: Number, required: true },
  selected: { type: Array, default: () => [] },
  showAnswer: { type: Boolean, default: false },
  mode: { type: String, default: 'sequential' },
  activeQuestions: { type: Array, default: () => [] },
  wrongIndexes: { type: Array, default: () => [] },
  visitedAt: { type: Object, default: () => ({}) }
})
const emit = defineEmits(['select', 'submit', 'next', 'prev', 'show', 'home', 'jump', 'visited'])

const showPanel = ref(false)

const typeLabel = computed(() => {
  if (props.question.type === 'single') return '单选题'
  if (props.question.type === 'multi') return '多选题'
  return '判断题'
})

const letters = computed(() => {
  if (props.question.type === 'judge') return ['A', 'B']
  // For single/multi: use only options that have content (A-F)
  const opts = props.question.options || {}
  return ['A', 'B', 'C', 'D', 'E', 'F'].filter(l => opts[l])
})

const isMulti = computed(() => props.question.type === 'multi')

function toggle(letter) {
  if (props.showAnswer) return
  if (isMulti.value) {
    const cur = [...props.selected]
    const idx = cur.indexOf(letter)
    if (idx >= 0) cur.splice(idx, 1)
    else cur.push(letter)
    emit('select', cur)
  } else {
    emit('select', [letter])
  }
}

function isSelected(letter) {
  return props.selected.includes(letter)
}

function isCorrect(letter) {
  return props.showAnswer && props.question.answer.includes(letter)
}

function isWrong(letter) {
  if (!props.showAnswer) return false
  return isSelected(letter) && !props.question.answer.includes(letter)
}

const progress = computed(() => {
  return Math.round(((props.index + 1) / props.total) * 100)
})

// Normalize answer: split into sorted unique letters array
function normalizeAnswer(ans) {
  if (!ans) return []
  return [...new Set(ans.toUpperCase().split(''))].filter(c => /[A-F]/.test(c)).sort()
}

const isAnswerCorrect = computed(() => {
  const userSorted = normalizeAnswer(props.selected.join(''))
  const correctSorted = normalizeAnswer(props.question.answer)
  if (userSorted.length !== correctSorted.length) return false
  return userSorted.every((l, i) => l === correctSorted[i])
})

const userAnswerText = computed(() => {
  const u = [...props.selected].sort().join('')
  return u || '（未选）'
})

// Build a quick-jump grid with memory tags
// Sort mode: 'sequential' | 'recent' | 'wrong-only'
const jumpSort = ref('sequential')
const jumpQuery = ref('')

const jumpList = computed(() => {
  const list = props.activeQuestions.map((q, idx) => {
    const id = q.id
    return {
      id, idx, q,
      isCurrent: idx === props.index,
      isWrong: props.wrongIndexes.includes(id),
      isVisited: !!props.visitedAt[id],
      visitedAt: props.visitedAt[id] || 0
    }
  })
  let filtered = list
  if (jumpQuery.value) {
    const q = jumpQuery.value.trim()
    if (/^\d+$/.test(q)) {
      // Numeric: jump directly to that question number (1-based)
      const n = parseInt(q, 10)
      filtered = list.filter(it => it.id + 1 === n)
    } else {
      filtered = list.filter(it => it.q.question.includes(q))
    }
  }
  if (jumpSort.value === 'recent') {
    filtered = [...filtered].sort((a, b) => b.visitedAt - a.visitedAt)
  } else if (jumpSort.value === 'wrong-only') {
    filtered = filtered.filter(it => it.isWrong)
  } else if (jumpSort.value === 'visited-only') {
    filtered = filtered.filter(it => it.isVisited)
  }
  return filtered.slice(0, 200) // cap for performance
})

const totalLabels = computed(() => ({
  total: props.activeQuestions.length,
  visited: Object.keys(props.visitedAt).filter(id => {
    return props.activeQuestions.some(q => String(q.id) === id)
  }).length,
  wrong: props.wrongIndexes.filter(id => props.activeQuestions.some(q => q.id === id)).length
}))

function doJump(targetId) {
  emit('jump', targetId)
  showPanel.value = false
  jumpQuery.value = ''
}

// Mark visited on mount and when question changes
function recordVisit() {
  emit('visited')
}

onMounted(() => {
  recordVisit()
  document.addEventListener('click', onDocClick)
})
onUnmounted(() => {
  document.removeEventListener('click', onDocClick)
})
function onDocClick(e) {
  if (!showPanel.value) return
  if (!e.target.closest('.q-toolbar')) {
    showPanel.value = false
  }
}
watch(() => props.question.id, () => {
  recordVisit()
})
</script>

<template>
  <div>
    <div class="topbar">
      <button class="back" @click="emit('home')">‹ 首页</button>
      <span class="title">{{ typeLabel }}</span>
      <span class="placeholder"></span>
    </div>
    <div class="progress-bar">
      <div class="progress-fill" :style="{ width: progress + '%' }"></div>
    </div>

    <div class="q-toolbar">
      <button class="tool-btn" @click="showPanel = !showPanel">
        🔢 选题 <span class="hint">第 {{ index + 1 }} / {{ total }}</span>
      </button>
      <div v-if="showPanel" class="jump-panel" @click.stop>
        <div class="jump-header">
          <input
            v-model="jumpQuery"
            class="jump-input"
            placeholder="输入题号或关键词搜索…"
          />
        </div>
        <div class="jump-tabs">
          <button :class="['tab', { active: jumpSort === 'sequential' }]" @click="jumpSort='sequential'">顺序</button>
          <button :class="['tab', { active: jumpSort === 'recent' }]" @click="jumpSort='recent'">最近访问</button>
          <button :class="['tab', { active: jumpSort === 'visited-only' }]" @click="jumpSort='visited-only'">已浏览</button>
          <button :class="['tab', { active: jumpSort === 'wrong-only' }]" @click="jumpSort='wrong-only'">⚠️ 错题 ({{ totalLabels.wrong }})</button>
        </div>
        <div class="jump-grid">
          <button
            v-for="item in jumpList"
            :key="item.idx"
            :class="['jump-num', { current: item.isCurrent, wrong: item.isWrong, visited: item.isVisited }]"
            @click="doJump(item.id)"
            :title="`第 ${item.id + 1} 题`"
          >
            <span class="num-text">{{ item.id + 1 }}</span>
            <span v-if="item.isWrong" class="badge wrong-badge">错</span>
            <span v-else-if="item.isVisited" class="badge visited-badge">●</span>
          </button>
          <div v-if="jumpList.length === 0" class="empty">无匹配题目</div>
        </div>
      </div>
    </div>

    <div class="question-page" style="padding-top:16px;">
      <div class="question-card">
        <div class="q-meta">第 {{ index + 1 }} / {{ total }} 题 · {{ typeLabel }}{{ isMulti ? '（可多选）' : '' }}</div>
        <div class="q-text">{{ question.question }}</div>

        <div v-if="question.type !== 'judge'" class="options">
          <button
            v-for="letter in letters"
            :key="letter"
            class="option"
            :class="{
              selected: isSelected(letter) && !showAnswer,
              correct: isCorrect(letter),
              wrong: isWrong(letter)
            }"
            @click="toggle(letter)"
          >
            <span class="letter">{{ letter }}</span>
            <span class="text">{{ question.options[letter] }}</span>
          </button>
        </div>

        <div v-else class="options">
          <button
            v-for="letter in letters"
            :key="letter"
            class="option"
            :class="{
              selected: isSelected(letter) && !showAnswer,
              correct: showAnswer && question.answer === letter,
              wrong: isWrong(letter)
            }"
            @click="toggle(letter)"
          >
            <span class="letter">{{ letter }}</span>
            <span class="text">{{ question.options[letter] }}</span>
          </button>
        </div>

        <div v-if="showAnswer" class="answer-feedback">
          <span class="label">正确答案：</span>
          <span style="font-weight:600;color:#2e7d32;">{{ question.answer }}</span>
          <span v-if="isMulti" style="margin-left:8px;font-size:12px;color:#757575;">（多选）</span>
          <div v-if="isAnswerCorrect" style="margin-top:6px;color:#4caf50;font-size:13px;">✓ 回答正确</div>
          <div v-else style="margin-top:6px;color:#f44336;font-size:13px;">
            ✗ 你的答案：{{ userAnswerText }}，已加入错题本
          </div>
        </div>
      </div>
    </div>

    <div class="bottom-bar">
      <button v-if="index > 0" class="btn-secondary" @click="emit('prev')">上一题</button>
      <button v-else class="btn-secondary" disabled>上一题</button>

      <button v-if="!showAnswer" class="btn-primary" @click="emit('submit')">提交答案</button>
      <button v-else class="btn-primary" @click="emit('next')">
        {{ index < total - 1 ? '下一题' : '查看结果' }}
      </button>
    </div>
  </div>
</template>

<style scoped>
.correct-text { color: #4caf50; }
.wrong-text { color: #f44336; }

/* Jump-to-question toolbar */
.q-toolbar {
  position: relative;
  padding: 8px 16px;
  background: #fafafa;
  border-bottom: 1px solid #ececec;
  display: flex;
  align-items: center;
  gap: 12px;
}
.tool-btn {
  background: #fff;
  border: 1px solid #ddd;
  padding: 6px 12px;
  border-radius: 6px;
  font-size: 13px;
  cursor: pointer;
  color: #424242;
}
.tool-btn .hint {
  margin-left: 6px;
  color: #757575;
  font-size: 12px;
}
.jump-panel {
  position: absolute;
  top: 100%;
  left: 16px;
  right: 16px;
  z-index: 100;
  background: #fff;
  border: 1px solid #ddd;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.15);
  max-height: 60vh;
  display: flex;
  flex-direction: column;
  margin-top: 4px;
}
.jump-header { padding: 8px; border-bottom: 1px solid #ececec; }
.jump-input {
  width: 100%;
  padding: 8px 10px;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 14px;
  box-sizing: border-box;
}
.jump-tabs {
  display: flex;
  border-bottom: 1px solid #ececec;
  background: #fafafa;
}
.jump-tabs .tab {
  flex: 1;
  background: transparent;
  border: none;
  padding: 8px 4px;
  font-size: 12px;
  cursor: pointer;
  color: #616161;
  border-bottom: 2px solid transparent;
}
.jump-tabs .tab.active {
  color: #1976d2;
  border-bottom-color: #1976d2;
  font-weight: 600;
}
.jump-grid {
  flex: 1;
  overflow-y: auto;
  padding: 8px;
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(44px, 1fr));
  gap: 6px;
}
.jump-num {
  position: relative;
  background: #f5f5f5;
  border: 1px solid #e0e0e0;
  border-radius: 6px;
  padding: 8px 4px;
  font-size: 13px;
  cursor: pointer;
  min-height: 36px;
  color: #424242;
}
.jump-num:hover { background: #e3f2fd; border-color: #1976d2; }
.jump-num.visited { background: #fff8e1; border-color: #ffcc02; }
.jump-num.wrong { background: #ffebee; border-color: #ef5350; color: #c62828; font-weight: 600; }
.jump-num.current { background: #1976d2; color: #fff; border-color: #1976d2; box-shadow: 0 0 0 2px rgba(25,118,210,0.3); }
.jump-num .badge {
  position: absolute;
  top: 1px;
  right: 2px;
  font-size: 9px;
  line-height: 1;
}
.jump-num .visited-badge { color: #f57f17; }
.jump-num.current .visited-badge { color: #fff; }
.jump-num .wrong-badge { color: #d32f2f; font-weight: 700; }
.jump-num.current .wrong-badge { color: #fff; }
.jump-num .num-text { display: block; }
.empty { grid-column: 1 / -1; text-align: center; padding: 24px; color: #9e9e9e; font-size: 13px; }
</style>
