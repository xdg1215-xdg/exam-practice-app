<script setup>
import { computed } from 'vue'

const props = defineProps({
  stats: { type: Object, required: true },
  questions: { type: Array, required: true },
  userAnswers: { type: Object, required: true },
  wrongIndexes: { type: Array, default: () => [] }
})
const emit = defineEmits(['restart', 'home', 'review-wrong'])

const grade = computed(() => {
  const s = props.stats.score
  if (s >= 90) return { text: '优秀', color: '#4caf50' }
  if (s >= 80) return { text: '良好', color: '#8bc34a' }
  if (s >= 60) return { text: '及格', color: '#ff9800' }
  return { text: '待加强', color: '#f44336' }
})

const wrongQuestions = computed(() => {
  const list = []
  props.questions.forEach((q, idx) => {
    const ua = props.userAnswers[idx]
    if (ua && ua.length > 0) {
      const userSorted = [...ua].sort().join('')
      const correctSorted = q.answer.split('').sort().join('')
      if (userSorted !== correctSorted) {
        list.push({ q, userAns: ua })
      }
    } else {
      list.push({ q, userAns: [] })
    }
  })
  return list
})
</script>

<template>
  <div>
    <div class="topbar">
      <button class="back" @click="emit('home')">‹ 首页</button>
      <span class="title">练习结果</span>
      <span class="placeholder"></span>
    </div>

    <div class="result-page">
      <div class="result-card">
        <div style="font-size:14px;color:#757575;">本次得分</div>
        <div class="result-score">{{ stats.score }}<span style="font-size:24px;color:#9e9e9e;"> 分</span></div>
        <div class="result-grade" :style="{ color: grade.color }">{{ grade.text }}</div>

        <div class="result-stats">
          <div class="result-stat">
            <div class="num">{{ stats.total }}</div>
            <div class="lbl">总题数</div>
          </div>
          <div class="result-stat">
            <div class="num" style="color:#4caf50;">{{ stats.correct }}</div>
            <div class="lbl">答对</div>
          </div>
          <div class="result-stat">
            <div class="num" style="color:#f44336;">{{ stats.wrong }}</div>
            <div class="lbl">答错</div>
          </div>
        </div>
      </div>

      <div class="result-actions">
        <button class="btn-primary" @click="emit('restart')">🔄 再来一次</button>
        <button v-if="stats.wrong > 0" class="btn-secondary" @click="emit('review-wrong')">❌ 练习错题 ({{ wrongQuestions.length }})</button>
        <button class="btn-secondary" @click="emit('home')">🏠 返回首页</button>
      </div>

      <div v-if="wrongQuestions.length > 0" style="text-align:left;margin-top:24px;">
        <h3 style="font-size:16px;margin-bottom:12px;color:#424242;">📋 错题回顾</h3>
        <div class="review-list">
          <div v-for="(item, i) in wrongQuestions" :key="i" class="review-item">
            <div class="q-num">第 {{ item.q.id + 1 }} 题 · 答错</div>
            <div class="q-text">{{ item.q.question }}</div>
            <div class="q-answer">
              <span class="your">你的答案：{{ item.userAns.length > 0 ? item.userAns.sort().join('') : '（未作答）' }}</span>
              &nbsp;&nbsp;
              <span class="correct">正确答案：{{ item.q.answer }}</span>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
