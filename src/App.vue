<script setup>
import { ref, computed, onMounted, shallowRef } from 'vue'
import QuestionView from './components/QuestionView.vue'
import ResultView from './components/ResultView.vue'
import HomeView from './components/HomeView.vue'

const view = ref('home') // home | practice | result
const mode = ref('sequential') // sequential | random | wrong
const questions = shallowRef([])
const userAnswers = ref({}) // qIndex -> array of selected letters
const currentIndex = ref(0)
const showAnswer = ref(false)
const wrongIndexes = ref([])
const finished = ref(false)

const currentQuestion = computed(() => {
  if (currentIndex.value >= activeQuestions.value.length) return null
  return activeQuestions.value[currentIndex.value]
})

const activeQuestions = ref([])
const originalIndexMap = ref({}) // for tracking original question index

onMounted(async () => {
  // Load questions
  const res = await import('./questions.json')
  questions.value = res.default
  // Load saved wrong questions
  const saved = localStorage.getItem('wrongQuestions')
  if (saved) {
    try {
      const idxs = JSON.parse(saved)
      wrongIndexes.value = idxs.filter(i => i < questions.value.length)
    } catch (e) {
      wrongIndexes.value = []
    }
  }
})

function startPractice(selectedMode) {
  mode.value = selectedMode
  if (selectedMode === 'sequential') {
    activeQuestions.value = [...questions.value]
  } else if (selectedMode === 'random') {
    activeQuestions.value = [...questions.value].sort(() => Math.random() - 0.5)
  } else if (selectedMode === 'wrong') {
    activeQuestions.value = wrongIndexes.value.map(i => questions.value[i])
    if (activeQuestions.value.length === 0) {
      alert('暂无错题，先去练习吧！')
      return
    }
  }
  originalIndexMap.value = activeQuestions.value.map(q => questions.value.indexOf(q))
  currentIndex.value = 0
  userAnswers.value = {}
  showAnswer.value = false
  finished.value = false
  view.value = 'practice'
}

function goHome() {
  view.value = 'home'
  finished.value = false
}

function onSelect(letters) {
  userAnswers.value[currentIndex.value] = letters
}

function onSubmit() {
  if (!userAnswers.value[currentIndex.value] || userAnswers.value[currentIndex.value].length === 0) {
    alert('请先选择答案')
    return
  }
  showAnswer.value = true
  // Check if wrong
  const q = currentQuestion.value
  const userAns = [...userAnswers.value[currentIndex.value]].sort().join('')
  const correctAns = q.answer.split('').sort().join('')
  if (userAns !== correctAns) {
    const origIdx = originalIndexMap.value[currentIndex.value]
    if (!wrongIndexes.value.includes(origIdx)) {
      wrongIndexes.value.push(origIdx)
      localStorage.setItem('wrongQuestions', JSON.stringify(wrongIndexes.value))
    }
  }
}

function onNext() {
  if (currentIndex.value < activeQuestions.value.length - 1) {
    currentIndex.value++
    showAnswer.value = false
  } else {
    // Finished
    finished.value = true
    view.value = 'result'
  }
}

function onPrev() {
  if (currentIndex.value > 0) {
    currentIndex.value--
    showAnswer.value = !!userAnswers.value[currentIndex.value]
  }
}

function onShowAnswer() {
  showAnswer.value = true
}

function clearWrong() {
  if (confirm('确定要清空所有错题吗？')) {
    wrongIndexes.value = []
    localStorage.removeItem('wrongQuestions')
  }
}

function restart() {
  startPractice(mode.value)
}

const stats = computed(() => {
  let total = activeQuestions.value.length
  let answered = 0
  let correct = 0
  activeQuestions.value.forEach((q, idx) => {
    const userAns = userAnswers.value[idx]
    if (userAns && userAns.length > 0) {
      answered++
      const userSorted = [...userAns].sort().join('')
      const correctSorted = q.answer.split('').sort().join('')
      if (userSorted === correctSorted) correct++
    }
  })
  return { total, answered, correct, wrong: answered - correct, score: total > 0 ? Math.round((correct / total) * 100) : 0 }
})
</script>

<template>
  <div class="container">
    <HomeView v-if="view === 'home'" :total="questions.length" :wrongCount="wrongIndexes.length" @start="startPractice" @clear="clearWrong" />
    <QuestionView
      v-else-if="view === 'practice' && currentQuestion"
      :question="currentQuestion"
      :index="currentIndex"
      :total="activeQuestions.length"
      :selected="userAnswers[currentIndex] || []"
      :showAnswer="showAnswer"
      :mode="mode"
      @select="onSelect"
      @submit="onSubmit"
      @next="onNext"
      @prev="onPrev"
      @show="onShowAnswer"
      @home="goHome"
    />
    <ResultView
      v-else-if="view === 'result'"
      :stats="stats"
      :questions="activeQuestions"
      :userAnswers="userAnswers"
      :wrongIndexes="originalIndexMap"
      @restart="restart"
      @home="goHome"
      @review-wrong="() => { startPractice('wrong') }"
    />
  </div>
</template>
