<script setup lang="ts">
import JsonNode from '@/components/JsonNode.vue'
import { reactive } from 'vue';

const props = defineProps<{ data: any }>()
const emit = defineEmits<{
  (e: 'update', data: any): void
  (e: 'copied'): void
}>()

// 用于存储每个节点的展开状态，键为路径（路径递归包含子节点）
const expandedKeys = reactive<Record<string, boolean>>({});

function copyTree() {
  if (props.data) {
    const jsonString = JSON.stringify(props.data, null, 2)
    navigator.clipboard.writeText(jsonString)
    emit('copied')
  }
}

function toggleExpand(path: string) {
  expandedKeys[path] = !expandedKeys[path]
}

// 展开所有节点
function expandAll() {
  if (!props.data) return

  // 递归展开所有节点
  const expandRecursive = (obj: any, currentPath: string) => {
    if (typeof obj === 'object' && obj !== null) {
      expandedKeys[currentPath] = true
      Object.keys(obj).forEach(key => {
        const childValue = obj[key]
        const newPath = `${currentPath}.${key}`

        if (typeof childValue === 'object' && childValue !== null) {
          expandRecursive(childValue, newPath)
        }
      });
    }
  }
  expandRecursive(props.data, 'root')
}

function collapseToFirstLevel() {
  if (!props.data) return;

  // 清空所有展开状态
  Object.keys(expandedKeys).forEach(key => {
    delete expandedKeys[key];
  });

  // 只展开第一层
  expandedKeys['root'] = true;
}

expandedKeys['root'] = true;
</script>

<template>
  <div class="right-panel">
    <div class="panel-header">
      <div class="panel-title">树形视图（可编辑）</div>
      <div class="buttons">
        <button class="expand-all" @click="expandAll">展开全部</button>
        <button class="collapse-first" @click="collapseToFirstLevel">收起至第一层</button>
        <button class="copy" @click="copyTree">复制JSON</button>
      </div>
    </div>
    <div class="output-section">
      <JsonNode :expandedKeys="expandedKeys" v-if="data" :value="data" path="root" @update="emit('update', $event)"
        @toggle="toggleExpand" />
      <div v-else class="empty">暂无数据</div>
    </div>
  </div>
</template>

<style scoped>
.right-panel {
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
  flex-shrink: 0;
}

.panel-title {
  font-weight: bold;
  color: #2c3e50;
}

.output-section {
  border: 1px solid #ddd;
  border-radius: 4px;
  padding: 12px;
  background-color: #f9f9f9;
  flex: 1;
  overflow: hidden;
  overflow-y: auto;
  min-height: 0;
}

.buttons {
  display: flex;
  gap: 10px;
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

button.expand-all {
  background-color: #f39c12;
}

button.expand-all:hover {
  background-color: #e67e22;
}

button.collapse-first {
  background-color: #95a5a6;
}

button.collapse-first:hover {
  background-color: #7f8c8d;
}

button.copy {
  background-color: #27ae60;
}

button.copy:hover {
  background-color: #229954;
}

button:hover {
  opacity: 0.9;
}

.empty {
  color: #7f8c8d;
  text-align: center;
  padding: 20px;
  font-style: italic;
}

@media (max-width: 768px) {
  .output-section {
    height: 400px;
    min-height: 400px;
  }

  .panel-header {
    flex-direction: column;
    align-items: stretch;
  }

  .buttons {
    justify-content: center;
  }

  button {
    flex: 1;
    min-width: 100px;
  }
}
</style>
