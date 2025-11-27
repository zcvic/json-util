<script setup lang="ts">
import JsonNode from '@/components/JsonNode.vue'
import JsonSearch from '@/components/JsonSearch.vue'
import { reactive, ref } from 'vue';

const props = defineProps<{ data: any }>()
const emit = defineEmits<{
  (e: 'update', data: any): void
  (e: 'copied'): void
}>()

const expandedKeys = reactive<Record<string, boolean>>({});
const searchResults = ref<Array<{ path: string; value: any; key: string; matchType?: 'key' | 'value' }>>([]);

function copyTree() {
  if (props.data) {
    const processedData = { ...props.data };
    
    Object.keys(processedData).forEach(key => {
      if (typeof processedData[key] === 'object' && processedData[key] !== null) {
        processedData[key] = JSON.stringify(processedData[key]);
      }
    });
    
    const jsonString = JSON.stringify(processedData, null, 2);
    navigator.clipboard.writeText(jsonString);
    emit('copied');
  }
}

function toggleExpand(path: string) {
  expandedKeys[path] = !expandedKeys[path]
}

function expandAll() {
  if (!props.data) return

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

  Object.keys(expandedKeys).forEach(key => {
    delete expandedKeys[key];
  });

  expandedKeys['root'] = true;
}

function addChild(path?: string) {
  if (!props.data) return;

  const addToPath = (obj: any, targetPath: string, newData: any): boolean => {
    if (targetPath === 'root') {
      if (typeof obj === 'object' && obj !== null) {
        let newKey = 'newKey';
        let counter = 1;
        while (obj.hasOwnProperty(newKey)) {
          newKey = `newKey${counter}`;
          counter++;
        }
        obj[newKey] = newData;
        return true;
      }
      return false;
    }

    const pathParts = targetPath.split('.').slice(1);
    let current = obj;
    
    for (let i = 0; i < pathParts.length; i++) {
      if (typeof current !== 'object' || current === null) {
        return false;
      }
      current = current[pathParts[i]];
    }
    
    if (typeof current === 'object' && current !== null) {
      let newKey = 'newKey';
      let counter = 1;
      while (current.hasOwnProperty(newKey)) {
        newKey = `newKey${counter}`;
        counter++;
      }
      current[newKey] = newData;
      return true;
    }
    
    return false;
  };

  const newData = '';
  const targetPath = path || 'root';
  const updatedData = JSON.parse(JSON.stringify(props.data));
  const success = addToPath(updatedData, targetPath, newData);
  
  if (success) {
    emit('update', updatedData);
    
    expandedKeys[targetPath] = true;
    
    if (targetPath !== 'root') {
      const pathParts = targetPath.split('.');
      const parentPath = pathParts.slice(0, -1).join('.') || 'root';
      expandedKeys[parentPath] = true;
    }
  }
}

function deleteChild(key: string | number) {
  if (!props.data) return;

  const updatedData = JSON.parse(JSON.stringify(props.data));
  let success = false;
  
  if (typeof key === 'string' && key.includes('.')) {
    success = deleteByPath(updatedData, key);
  } else {
    const keyStr = String(key);
    if (Array.isArray(updatedData)) {
      const index = Number(keyStr);
      if (!isNaN(index) && index >= 0 && index < updatedData.length) {
        updatedData.splice(index, 1);
        success = true;
      }
    } else {
      if (keyStr in updatedData) {
        delete updatedData[keyStr];
        success = true;
      }
    }
  }
  
  if (success) {
    emit('update', updatedData);
  }
}

function deleteByPath(obj: any, path: string) {
  if (path === 'root') return false;
  
  const pathParts = path.split('.').slice(1);
  let current = obj;
  
  for (let i = 0; i < pathParts.length - 1; i++) {
    if (typeof current !== 'object' || current === null || !(pathParts[i] in current)) {
      return false;
    }
    current = current[pathParts[i]];
  }
  
  const lastKey = pathParts[pathParts.length - 1];
  if (Array.isArray(current)) {
    const index = parseInt(lastKey);
    if (!isNaN(index) && index >= 0 && index < current.length) {
      current.splice(index, 1);
      return true;
    }
  } else if (typeof current === 'object' && current !== null && lastKey in current) {
    delete current[lastKey];
    return true;
  }
  
  return false;
}

function updateJsonData(updateData: any) {
  if (!props.data) return;

  if (typeof updateData === 'object' && updateData !== null && 'path' in updateData) {
    const { path, key, value } = updateData;
    const updatedData = JSON.parse(JSON.stringify(props.data));
    
    function isSerializableJsonString(value: unknown): boolean {
      if (typeof value !== 'string') return false;
      try {
        const parsed = JSON.parse(value);
        return typeof parsed === 'object' && parsed !== null;
      } catch {
        return false;
      }
    }
    
    const navigateAndUpdate = (obj: any, targetPath: string, targetKey: string, newValue: any) => {
      if (targetPath === 'root') {
        if (targetKey === '') {
          return newValue;
        } else {
          obj[targetKey] = newValue;
          return obj;
        }
      } else {
        const pathParts = targetPath.split('.').slice(1);
        let current = obj;
        
        for (let i = 0; i < pathParts.length - 1; i++) {
          if (typeof current !== 'object' || current === null) {
            return obj;
          }
          current = current[pathParts[i]];
        }
        
        const parent = current;
        const lastKey = pathParts[pathParts.length - 1];
        
        if (targetKey === '') {
          parent[lastKey] = newValue;
        } else {
          parent[lastKey][targetKey] = newValue;
        }
        
        return obj;
      }
    };
    
    const result = navigateAndUpdate(updatedData, path, key, value);
    emit('update', result);
  } else {
    emit('update', updateData);
  }
}

function addSibling(parentPath: string, index: number) {
  if (!props.data) return;

  const addSiblingToPath = (obj: any, targetPath: string, insertIndex: number, newData: any): boolean => {
    if (targetPath === 'root') {
      if (typeof obj === 'object' && obj !== null && !Array.isArray(obj)) {
        let newKey = 'newKey';
        let counter = 1;
        while (obj.hasOwnProperty(newKey)) {
          newKey = `newKey${counter}`;
          counter++;
        }
        obj[newKey] = newData;
        return true;
      }
      return false;
    }

    const pathParts = targetPath.split('.').slice(1);
    let current = obj;
    
    for (let i = 0; i < pathParts.length; i++) {
      if (typeof current !== 'object' || current === null) {
        return false;
      }
      current = current[pathParts[i]];
    }
    
    if (Array.isArray(current)) {
      current.splice(insertIndex, 0, newData);
    } else if (typeof current === 'object' && current !== null) {
      let newKey = 'newKey';
      let counter = 1;
      while (current.hasOwnProperty(newKey)) {
        newKey = `newKey${counter}`;
        counter++;
      }
      current[newKey] = newData;
    }
    
    return true;
  };

  const newData = '';
  const updatedData = JSON.parse(JSON.stringify(props.data));
  const success = addSiblingToPath(updatedData, parentPath, index, newData);
  
  if (success) {
    emit('update', updatedData);
    
    if (parentPath !== 'root') {
      expandedKeys[parentPath] = true;
    } else {
      expandedKeys['root'] = true;
    }
  }
}

// 搜索功能
function searchJson(payload: { query: string; type: 'all' | 'keys' | 'values'; useRegex: boolean }) {
  const { query, type, useRegex } = payload;
  const results: Array<{ path: string; value: any; key: string; matchType?: 'key' | 'value' }> = [];

  if (!query || !props.data) {
    searchResults.value = results;
    return;
  }

  // 递归搜索函数
  const searchRecursive = (obj: any, currentPath: string, parentKey?: string) => {
    if (typeof obj === 'object' && obj !== null) {
      // 搜索键名
      if (type === 'all' || type === 'keys') {
        Object.keys(obj).forEach(key => {
          if (useRegex) {
            try {
              const regex = new RegExp(query, 'gi');
              if (regex.test(key)) {
                results.push({
                  path: `${currentPath}.${key}`,
                  value: obj[key],
                  key,
                  matchType: 'key'
                });
              }
            } catch (e) {
              // 正则表达式无效，忽略
            }
          } else {
            if (key.toLowerCase().includes(query.toLowerCase())) {
              results.push({
                path: `${currentPath}.${key}`,
                value: obj[key],
                key,
                matchType: 'key'
              });
            }
          }
        });
      }

      // 搜索值
      if (type === 'all' || type === 'values') {
        Object.entries(obj).forEach(([key, value]) => {
          const valueStr = String(value);
          let matched = false;

          if (useRegex) {
            try {
              const regex = new RegExp(query, 'gi');
              matched = regex.test(valueStr);
            } catch (e) {
              // 正则表达式无效，忽略
            }
          } else {
            matched = valueStr.toLowerCase().includes(query.toLowerCase());
          }

          if (matched) {
            results.push({
              path: `${currentPath}.${key}`,
              value,
              key,
              matchType: 'value'
            });
          }

          // 递归搜索子对象
          if (typeof value === 'object' && value !== null) {
            searchRecursive(value, `${currentPath}.${key}`, key);
          }
        });
      }
    }
  };

  // 开始搜索
  searchRecursive(props.data, 'root');
  searchResults.value = results;
}

// 清空搜索结果
function clearSearch() {
  searchResults.value = [];
}

// 导航到搜索结果
function navigateToResult(result: { path: string; value: any; key: string; matchType?: 'key' | 'value' }) {
  // 展开所有父节点
  const pathParts = result.path.split('.');
  for (let i = 1; i < pathParts.length; i++) {
    const parentPath = pathParts.slice(0, i).join('.');
    expandedKeys[parentPath] = true;
  }
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
    <JsonSearch 
      :data="data" 
      :searchResults="searchResults"
      @search="searchJson"
      @clear="clearSearch"
      @navigate="navigateToResult"
    />
    <div class="output-section">
      <JsonNode 
        :expandedKeys="expandedKeys" 
        v-if="data" 
        :value="data" 
        path="root" 
        :isRoot="true"
        @update="updateJsonData($event)"
        @toggle="toggleExpand"
        @addChild="addChild"
        @deleteChild="deleteChild"
        @addSibling="addSibling"
      />
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

