<script setup lang="ts">
import { ref, watch } from 'vue'

const props = defineProps<{ data: any }>()
const emit = defineEmits<{
  (e: 'update', data: any): void
  (e: 'copied'): void
}>()

const inputValue = ref('')

// 监听外部数据变化，同步到输入框
watch(
  () => props.data,
  (newVal) => {
    if (newVal) {
      inputValue.value = JSON.stringify(newVal, null, 2)
    }
  },
  { immediate: true }
)

function formatJson() {
  try {
    const parsed = JSON.parse(inputValue.value.trim())
    // try {
    //     parsed = JSON.parse(parsed)
    // } catch (err) { console.log(err) }
    emit('update', parsed)
    inputValue.value = JSON.stringify(parsed, null, 2)
  } catch (err: unknown) {
    if (err instanceof Error) {
      alert('错误: ' + err.message)
    } else {
      alert('未知错误')
    }
  }
}

function copyInput() {
  navigator.clipboard.writeText(inputValue.value)
  emit('copied')
}

function clearInput() {
  inputValue.value = ''
  emit('update', '')
}
</script>

<template>
  <div class="left-panel">
    <div class="panel-header">
      <div class="panel-title">输入JSON</div>
      <div class="buttons">
        <button @click="formatJson">格式化</button>
        <button class="copy" @click="copyInput">复制</button>
        <button class="clear" @click="clearInput">清空</button>
      </div>
    </div>
    <textarea v-model="inputValue" placeholder="请输入JSON字符串..."></textarea>
  </div>
</template>

<style scoped>
.left-panel {
  flex: 1;
  display: flex;
  flex-direction: column;
  height: 80vh;
  min-height: 500px;
}

.panel-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}

.panel-title {
  font-weight: bold;
  color: #2c3e50;
}

textarea {
  flex: 1;
  padding: 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-family: monospace;
  font-size: 14px;
  resize: none;
  min-height: 400px;
  overflow-y: auto;
}

.buttons {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

button {
  padding: 8px 12px;
  background-color: #3498db;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
  transition: background-color 0.3s;
  font-size: 14px;
}

button.copy {
  background-color: #27ae60;
}

button.clear {
  background-color: #e74c3c;
}

button:not(.copy):not(.clear) {
  background-color: #3498db;
}

@media (max-width: 768px) {
  .left-panel {
    height: 400px;
    min-height: 400px;
  }

  textarea {
    min-height: 300px;
  }
}
</style>
