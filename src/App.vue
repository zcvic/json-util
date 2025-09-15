<script setup lang="ts">
import { ref } from 'vue'
import JsonInput from '@/components/JsonInput.vue'
import JsonTree from '@/components/JsonTree.vue'
import CopyNotification from '@/components/CopyNotification.vue'

const jsonData = ref<any>("")
const showNotification = ref(false)

// 更新 JSON
function updateJson(newJson: any) {
  jsonData.value = newJson
}

// 显示复制提示
function triggerNotification() {
  showNotification.value = true
  setTimeout(() => (showNotification.value = false), 2000)
}
</script>

<template>
  <div class="container">
    <h1>JSON树形格式化工具</h1>

    <div class="layout">
      <JsonInput :data="jsonData" @update="updateJson" @copied="triggerNotification" />
      <JsonTree :data="jsonData" @update="updateJson" @copied="triggerNotification" />
    </div>

    <CopyNotification v-if="showNotification" />
  </div>
</template>

<style scoped>
.container {
  max-width: 90%;
  margin: 0 auto;
  background-color: #fff;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  padding: 20px;
}

h1 {
  text-align: center;
}

.layout {
  display: flex;
  gap: 0px 50px;
  min-height: 500px;
}

@media (max-width: 900px) {
  .layout {
    flex-direction: column;
  }
}
</style>
