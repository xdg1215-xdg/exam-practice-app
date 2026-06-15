<script setup>
import { computed } from 'vue'

const props = defineProps({
  question: { type: Object, required: true },
  index: { type: Number, required: true },
  total: { type: Number, required: true },
  selected: { type: Array, default: () => [] },
  showAnswer: { type: Boolean, default: false },
  mode: { type: String, default: 'sequential' }
})
const emit = defineEmits(['select', 'submit', 'next', 'prev', 'show', 'home'])

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
</style>
