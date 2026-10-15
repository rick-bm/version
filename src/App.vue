<script setup>
import { ref, onMounted } from 'vue'

const version = ref('')
const loading = ref(true)
const error = ref('')

onMounted(async () => {
  try {
    const res = await fetch('/api/version')
    if (!res.ok) throw new Error(`HTTP ${res.status}`)
    const data = await res.json()
    if (typeof data.version !== 'string' || data.version === '') {
      throw new Error('missing version field')
    }
    version.value = data.version
  } catch {
    error.value = '获取版本号失败'
  } finally {
    loading.value = false
  }
})
</script>

<template>
  <main>
    <h1>版本号</h1>
    <p v-if="loading">加载中…</p>
    <p v-else-if="error">{{ error }}</p>
    <p v-else>{{ version }}</p>
  </main>
</template>
