<script setup>
defineProps({
  total: { type: Number, default: 0 },
  wrongCount: { type: Number, default: 0 }
})
const emit = defineEmits(['start', 'clear'])
</script>

<template>
  <div class="home">
    <h1>📚 常化制度练习</h1>
    <p class="subtitle">中盐常化 2026 年职能部室中层管理人员制度学习题库</p>

    <div class="mode-list">
      <button class="mode-btn" @click="emit('start', 'sequential')">
        <span class="title">📖 顺序练习</span>
        <span class="desc">共 {{ total }} 题，按顺序逐题练习</span>
      </button>
      <button class="mode-btn" @click="emit('start', 'random')">
        <span class="title">🎲 随机练习</span>
        <span class="desc">随机抽题，模拟真实考试</span>
      </button>
      <button class="mode-btn" @click="emit('start', 'wrong')">
        <span class="title">❌ 错题重练</span>
        <span class="desc">当前已收藏 {{ wrongCount }} 道错题</span>
      </button>
    </div>

    <div v-if="wrongCount > 0" class="stats">
      <button class="btn-secondary" style="margin-top:16px;padding:8px 16px;border-radius:6px;font-size:13px;" @click="emit('clear')">
        清空错题本
      </button>
    </div>

    <p class="stats">
      单选题 {{ total }} 道 · 多选/判断题混合<br>
      每题答完自动显示正确答案
    </p>
  </div>
</template>
