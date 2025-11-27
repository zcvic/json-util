<script setup lang="ts">
import { ref, watch } from 'vue'

// 防抖函数
function debounce<T extends (...args: any[]) => void>(
  func: T,
  wait: number
): (...args: Parameters<T>) => void {
  let timeout: number | null = null
  return (...args: Parameters<T>) => {
    if (timeout) clearTimeout(timeout)
    timeout = window.setTimeout(() => func(...args), wait)
  }
}

const props = defineProps<{
  data: any
  searchResults: Array<{ path: string; value: any; key: string; matchType?: 'key' | 'value' }>
}>()

const emit = defineEmits<{
  (e: 'search', payload: { query: string; type: 'all' | 'keys' | 'values'; useRegex: boolean }): void
  (e: 'clear'): void
  (e: 'navigate', result: { path: string; value: any; key: string; matchType?: 'key' | 'value' }): void
}>()

const searchQuery = ref('')
const isSearching = ref(false)
const currentResultIndex = ref(0)
const showResults = ref(false)
const searchType = ref<'all' | 'keys' | 'values'>('all') // 搜索类型：全部/键名/值
const searchTime = ref(0) // 搜索耗时
const lastQuery = ref('') // 记录上一次搜索
const useRegex = ref(false) // 是否使用正则表达式
const showSearchSettings = ref(false) // 是否显示搜索设置

// 搜索功能
function performSearch() {
  if (!searchQuery.value.trim()) {
    emit('clear')
    showResults.value = false
    return
  }
  
  const startTime = Date.now()
  isSearching.value = true
  showResults.value = true
  currentResultIndex.value = 0
  
  emit('search', { query: searchQuery.value, type: searchType.value, useRegex: useRegex.value })
  
  searchTime.value = Date.now() - startTime
  lastQuery.value = searchQuery.value
  isSearching.value = false
}

// 导航到结果
function navigateToResult(result: { path: string; value: any; key: string; matchType?: 'key' | 'value' }) {
  emit('navigate', result)
  showResults.value = false
}

// 键盘导航
function handleKeydown(event: KeyboardEvent) {
  if (!showResults.value || props.searchResults.length === 0) return
  
  switch (event.key) {
    case 'ArrowDown':
      event.preventDefault()
      currentResultIndex.value = (currentResultIndex.value + 1) % props.searchResults.length
      break
    case 'ArrowUp':
      event.preventDefault()
      currentResultIndex.value = currentResultIndex.value === 0 
        ? props.searchResults.length - 1 
        : currentResultIndex.value - 1
      break
    case 'Enter':
      event.preventDefault()
      if (props.searchResults[currentResultIndex.value]) {
        navigateToResult(props.searchResults[currentResultIndex.value])
      }
      break
    case 'Escape':
      showResults.value = false
      break
  }
}

// 清空搜索
function clearSearch() {
  searchQuery.value = ''
  emit('clear')
  showResults.value = false
}

// 创建防抖搜索函数
const debouncedSearch = debounce((query: string) => {
  if (query.trim()) {
    performSearch()
  } else {
    clearSearch()
  }
}, 300)

// 监听搜索输入
watch(searchQuery, (newQuery) => {
  debouncedSearch(newQuery)
})

// 格式化路径显示
function formatPath(path: string): string {
  return path.split('.').join(' → ')
}

// 高亮搜索结果
function highlightText(text: string, query: string): string {
  if (!query) return text
  const regex = new RegExp(`(${query})`, 'gi')
  return text.replace(regex, '<mark>$1</mark>')
}
</script>

<template>
  <div class="search-container">
    <div class="search-header">
      <div class="search-input-wrapper">
        <input
          v-model="searchQuery"
          type="text"
          placeholder="搜索JSON数据..."
          class="search-input"
          @keydown="handleKeydown"
          @focus="showResults = true"
        />
        <button 
          v-if="searchQuery" 
          @click="clearSearch" 
          class="clear-btn"
          title="清空搜索"
        >
          ✕
        </button>
        <button 
          @click="showSearchSettings = !showSearchSettings" 
          class="settings-btn"
          :class="{ active: showSearchSettings }"
          title="搜索设置"
        >
          ⚙️
        </button>
        <div v-if="isSearching" class="search-loading">🔍</div>
      </div>
      
      <!-- 搜索类型选择 -->
      <div class="search-controls" v-show="showSearchSettings">
        <div class="search-type-selector">
          <label class="search-type-label">搜索范围:</label>
          <div class="search-type-options">
            <label class="search-type-option">
              <input
                v-model="searchType"
                type="radio"
                value="all"
              />
              <span>全部</span>
            </label>
            <label class="search-type-option">
              <input
                v-model="searchType"
                type="radio"
                value="keys"
              />
              <span>键名</span>
            </label>
            <label class="search-type-option">
              <input
                v-model="searchType"
                type="radio"
                value="values"
              />
              <span>值</span>
            </label>
          </div>
        </div>
        
        <!-- 正则表达式选项 -->
        <label class="regex-option">
          <input
            v-model="useRegex"
            type="checkbox"
          />
          <span>正则表达式</span>
        </label>
      </div>
    </div>
    
    <div v-if="showResults && searchResults.length > 0" class="search-results">
      <div class="results-header">
        找到 {{ searchResults.length }} 个结果
      </div>
      <div class="results-list">
        <div
          v-for="(result, index) in searchResults"
          :key="result.path"
          class="result-item"
          :class="{ active: index === currentResultIndex }"
          @click="navigateToResult(result)"
        >
          <div class="result-path">{{ formatPath(result.path) }}</div>
          <div class="result-content">
            <div class="result-key" v-if="result.key && result.key !== result.path">
              <span class="key-label">键:</span>
              <span class="key-value" v-html="highlightText(String(result.key), searchQuery)"></span>
            </div>
            <div class="result-value" v-html="highlightText(String(result.value), searchQuery)"></div>
          </div>
          <div class="result-type" v-if="result.matchType">
            <span class="match-type-badge" :class="result.matchType">{{ result.matchType === 'key' ? '键名匹配' : '值匹配' }}</span>
          </div>
        </div>
      </div>
    </div>
    
    <div v-else-if="showResults && searchQuery.trim()" class="no-results">
      未找到匹配的结果
    </div>
  </div>
</template>

<style scoped>
.search-container {
  position: relative;
  margin-bottom: 20px;
}

.search-header {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.search-input-wrapper {
  position: relative;
  display: flex;
  align-items: center;
}

.search-input {
  width: 100%;
  padding: 12px 75px 12px 15px;
  border: 2px solid #e1e5e9;
  border-radius: 8px;
  font-size: 14px;
  outline: none;
  transition: all 0.3s ease;
  background-color: #fff;
}

.search-input:focus {
  border-color: #3498db;
  box-shadow: 0 0 0 3px rgba(52, 152, 219, 0.1);
}

.search-controls {
  display: flex;
  align-items: center;
  gap: 25px;
  flex-wrap: wrap;
  padding: 12px 15px;
  background-color: #f8f9fa;
  border: 1px solid #e1e5e9;
  border-radius: 8px;
  margin-top: 8px;
  transition: all 0.3s ease;
}

.search-type-selector {
  display: flex;
  align-items: center;
  gap: 15px;
  flex-wrap: wrap;
}

.search-type-label {
  font-size: 13px;
  font-weight: 600;
  color: #4a5568;
  white-space: nowrap;
  background-color: #e2e8f0;
  padding: 4px 10px;
  border-radius: 6px;
}

.search-type-options {
  display: flex;
  gap: 20px;
  background-color: white;
  padding: 8px 12px;
  border-radius: 6px;
  border: 1px solid #e2e8f0;
}

.search-type-option {
  display: flex;
  align-items: center;
  gap: 6px;
  cursor: pointer;
  font-size: 13px;
  color: #4a5568;
  transition: all 0.3s ease;
  padding: 4px 8px;
  border-radius: 4px;
}

.search-type-option:hover {
  color: #3498db;
  background-color: rgba(52, 152, 219, 0.1);
}

.search-type-option input[type="radio"] {
  margin: 0;
  cursor: pointer;
  accent-color: #3498db;
  width: 16px;
  height: 16px;
  transition: all 0.2s ease;
}

.search-type-option input[type="radio"]:checked {
  transform: scale(1.1);
}

.search-type-option span {
  user-select: none;
  font-weight: 500;
}

.regex-option {
  display: flex;
  align-items: center;
  gap: 6px;
  cursor: pointer;
  font-size: 13px;
  color: #4a5568;
  transition: all 0.3s ease;
  padding: 8px 12px;
  background-color: white;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  font-weight: 500;
}

.regex-option:hover {
  color: #3498db;
  background-color: rgba(52, 152, 219, 0.1);
  border-color: #3498db;
}

.regex-option input[type="checkbox"] {
  margin: 0;
  cursor: pointer;
  accent-color: #3498db;
  width: 16px;
  height: 16px;
  transition: all 0.2s ease;
}

.regex-option input[type="checkbox"]:checked {
  transform: scale(1.1);
}

.clear-btn {
  position: absolute;
  right: 70px;
  background: none;
  border: none;
  color: #999;
  cursor: pointer;
  font-size: 16px;
  padding: 5px;
  border-radius: 50%;
  transition: all 0.3s ease;
}

.clear-btn:hover {
  background-color: #f5f5f5;
  color: #333;
}

.settings-btn {
  position: absolute;
  right: 40px;
  background: none;
  border: none;
  color: #999;
  cursor: pointer;
  font-size: 16px;
  padding: 5px;
  border-radius: 50%;
  transition: all 0.3s ease;
  transform: rotate(0deg);
}

.settings-btn:hover {
  background-color: #f5f5f5;
  color: #3498db;
  transform: rotate(15deg);
}

.settings-btn.active {
  color: #3498db;
  transform: rotate(45deg);
  background-color: rgba(52, 152, 219, 0.1);
}

.settings-btn.active:hover {
  transform: rotate(60deg);
}

.search-loading {
  position: absolute;
  right: 15px;
  color: #3498db;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

.search-results {
  position: absolute;
  top: 100%;
  left: 0;
  right: 0;
  background: white;
  border: 1px solid #e1e5e9;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  max-height: 300px;
  overflow-y: auto;
  z-index: 1000;
  margin-top: 5px;
}

.results-header {
  padding: 10px 15px;
  background-color: #f8f9fa;
  border-bottom: 1px solid #e1e5e9;
  font-size: 12px;
  color: #666;
  font-weight: 500;
}

.results-list {
  max-height: 250px;
  overflow-y: auto;
}

.result-item {
  padding: 12px 15px;
  border-bottom: 1px solid #f0f0f0;
  cursor: pointer;
  transition: background-color 0.2s ease;
  display: flex;
  align-items: flex-start;
  gap: 8px;
}

.result-item:hover,
.result-item.active {
  background-color: #f8f9fa;
}

.result-item:last-child {
  border-bottom: none;
}

.result-path {
  font-size: 11px;
  color: #666;
  margin-bottom: 4px;
  font-family: monospace;
}

.result-value {
  font-size: 13px;
  color: #333;
  word-break: break-all;
}

.result-content {
  display: flex;
  flex-direction: column;
  gap: 4px;
  flex: 1;
}

.result-key {
  display: flex;
  align-items: center;
  gap: 4px;
}

.key-label {
  font-size: 11px;
  color: #666;
  font-weight: 500;
}

.key-value {
  font-size: 12px;
  color: #2c3e50;
  font-family: monospace;
}

.result-type {
  margin-left: 8px;
}

.match-type-badge {
  padding: 2px 6px;
  border-radius: 3px;
  font-size: 10px;
  font-weight: 500;
  text-transform: uppercase;
}

.match-type-badge.key {
  background-color: #e8f5e8;
  color: #2d5a2d;
}

.match-type-badge.value {
  background-color: #e8f4fd;
  color: #2d5a7a;
}

.result-value mark {
  background-color: #ffeb3b;
  color: #333;
  padding: 0 2px;
  border-radius: 2px;
}

.no-results {
  position: absolute;
  top: 100%;
  left: 0;
  right: 0;
  background: white;
  border: 1px solid #e1e5e9;
  border-radius: 8px;
  padding: 20px;
  text-align: center;
  color: #666;
  margin-top: 5px;
}

@media (max-width: 768px) {
  .search-container {
    margin-bottom: 15px;
  }
  
  .search-header {
    gap: 8px;
  }
  
  .search-type-selector {
    flex-direction: column;
    align-items: flex-start;
    gap: 8px;
  }
  
  .search-type-options {
    gap: 10px;
  }
  
  .search-results {
    max-height: 200px;
  }
  
  .results-list {
    max-height: 150px;
  }
  
  .result-item {
    padding: 10px 12px;
  }
  
  .result-path {
    font-size: 10px;
  }
  
  .result-value {
    font-size: 12px;
  }
  
  .key-value {
    font-size: 11px;
  }
}
</style>