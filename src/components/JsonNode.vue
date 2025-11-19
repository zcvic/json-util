<script setup lang="ts">
import { computed, ref } from 'vue';


const props = defineProps<{ 
  value: unknown; 
  path: string; 
  expandedKeys: Record<string, boolean>; 
  isRoot?: boolean;
  data?: any;
}>()
const emit = defineEmits<{
  (e: 'update', newData: unknown): void,
  (e: 'toggle', key: string): void;
  (e: 'addChild', path?: string): void;
  (e: 'deleteChild', key: string | number): void;
  (e: 'addSibling', parentPath: string, index: number): void;
}>()


const isExpanded = computed(() => props.expandedKeys[props.path] ?? false)

function tmpVal(v: unknown) {
  let tmp = v
  try {
    if (typeof v === 'string') {
      tmp = JSON.parse(v)
    }
  } catch {
    tmp = v
  }
  return tmp
}

function toggleExpand(key: string) {
  emit('toggle', key)
}

function updateChild(key: string | number, newValue: unknown) {
  const copy: Record<string | number, unknown> = { ...props.value as object }
  copy[key] = newValue
  emit('update', copy)
}

function updateChildKey(oldKey: string | number, newKey: string | number) {
  const obj = props.value as Record<string | number, unknown>
  if (oldKey === newKey) return
  
  const copy = { ...obj }
  const value = copy[oldKey]
  delete copy[oldKey]
  copy[newKey] = value
  emit('update', copy)
}

function onEdit(event: Event) {
  const el = event.target as HTMLElement | null
  if (!el) return
  
  const text = el.innerText.trim()
  let newValue: unknown = text

  if (/^".*"$/.test(text)) {
    newValue = text.slice(1, -1)
  } else if (/^\d+$/.test(text)) {
    newValue = Number(text)
  } else if (text === 'true' || text === 'false') {
    newValue = text === 'true'
  } else if (text === 'null') {
    newValue = null
  }

    emit('update', newValue)
}

function onKeyEdit(event: Event, oldKey: string | number) {
  const el = event.target as HTMLElement | null
  if (!el) return
  
  const text = el.innerText.trim()
  
  if (text === '' || text === oldKey.toString()) return
  
  const obj = props.value as Record<string | number, unknown>
  if (text in obj) {
    el.innerText = oldKey.toString()
    return
  }
  
  updateChildKey(oldKey, text)
}

function valColor(val: unknown) {
  const type = typeof val
  switch (type) {
    case 'string':
      return 'json-string'
    case 'number':
      return 'json-number'
    case 'boolean':
      return 'json-boolean'
    case 'object':
      return ''
    default:
      return 'json-null'
  }
}

function getPreviewValue(value: unknown): string {
  if (typeof value !== 'object' || value === null) {
    return String(value)
  }

  const obj = value as Record<string, unknown>
  const array = Array.isArray(value) ? (value as unknown[]) : []
  const isArray = Array.isArray(value)

  if (isArray) {
    const length = array.length
    if (length === 0) return '[]'
    if (length === 1) return `[ ${getPreviewValue(array[0])} ]`
    if (length === 2) return `[ ${getPreviewValue(array[0])}, ${getPreviewValue(array[1])} ]`
    return `[ ${getPreviewValue(array[0])}, ${getPreviewValue(array[1])}, ... ]`
  } else {
    const keys = Object.keys(obj)
    const length = keys.length
    if (length === 0) return '{}'
    if (length === 1) {
      const key = keys[0]
      return `{ ${key}: ${getPreviewValue(obj[key])} }`
    }
    if (length === 2) {
      const [k1, k2] = keys
      return `{ ${k1}: ${getPreviewValue(obj[k1])}, ${k2}: ${getPreviewValue(obj[k2])} }`
    }
    return `{ ${keys[0]}: ${getPreviewValue(obj[keys[0]])}, ${keys[1]}: ${getPreviewValue(obj[keys[1]])}, ... }`
  }
}



function addChild() {
  emit('addChild', props.path)
}

function addSibling() {
  const pathParts = props.path.split('.')
  if (pathParts.length > 1) {
    const parentPath = pathParts.slice(0, -1).join('.')
    const currentKey = pathParts[pathParts.length - 1]
    
    const parentIndex = parseInt(currentKey) || 0
    emit('addSibling', parentPath, parentIndex + 1)
  }
}

function getLastKey(path: string): string | number {
  const parts = path.split('.')
  const last = parts[parts.length - 1]
  const parsed = parseInt(last)
  return isNaN(parsed) ? last : parsed
}

function deleteNode() {
  if (!props.isRoot) {
    emit('deleteChild', getLastKey(props.path))
  }
}
</script>

<template>
  <div class="json-item">
    <template v-if="typeof value === 'object' && value !== null">
      <div class="node-header">
        <span class="expand-toggle" @click="toggleExpand(path)">{{ !isExpanded ? '▶' : '▼' }}</span>
        <span class="json-punctuation">{{ Array.isArray(value) ? '[' : '{' }}</span>

        <!-- 预览值显示 -->
        <span v-show="!isExpanded" class="preview-value">
          {{ getPreviewValue(value) }}
        </span>

        <!-- 操作按钮 -->
        <div class="node-actions">
          <button class="action-btn add-btn" @click="addChild" title="添加子节点">+</button>
          <button class="action-btn add-sibling-btn" @click="addSibling" title="添加同级节点" v-if="!props.isRoot">++</button>
          <button v-if="!isRoot" class="action-btn delete-btn" @click="deleteNode" title="删除节点">−</button>
        </div>
      </div>



      <div v-show="isExpanded">
        <div v-for="(v, k) in value" :key="k" class="json-item item">
          <div class="item-content">
            <span class="json-key editable-key" contenteditable="true" @blur="(e) => onKeyEdit(e, k)" @keydown.enter="(e) => { const target = e.target as HTMLElement | null; if (target) { target.blur(); } }">{{ k }}</span>
            <span class="json-punctuation">:</span>
            <JsonNode
              :expandedKeys="expandedKeys"
              :class="valColor(v)"
              :value="tmpVal(v)"
              :path="`${path}.${k}`"
              @update="updateChild(k, $event)"
              @toggle="toggleExpand"
              @addChild="(path) => emit('addChild', path)"
              @deleteChild="(path) => emit('deleteChild', path)"
              @addSibling="(parentPath, idx) => emit('addSibling', parentPath, idx)"
            />
          </div>
        </div>
      </div>

      <span class="json-punctuation closing-bracket">{{ Array.isArray(value) ? ']' : '}' }}</span>
    </template>

    <template v-else>
      <div class="leaf-node">
        <span class="editable" contenteditable="true" @blur="onEdit">{{ value }}</span>
        <div class="leaf-actions">
          <button v-if="!isRoot" class="action-btn delete-btn leaf-delete" @click="deleteNode" title="删除节点">−</button>
          <button class="action-btn add-sibling-btn leaf-add-sibling" @click="addSibling" title="添加同级节点" v-if="!props.isRoot">++</button>
        </div>
      </div>
    </template>
  </div>
</template>

<style scoped>
.json-item {
  display: block;
  margin: 10px 0;
  font-family: monospace;
  font-size: 14px;
  position: relative;
}

.node-header {
  display: flex;
  align-items: center;
  gap: 4px;
  margin-bottom: 8px;
}

.item {
  margin: 8px 0;
}

.item-content {
  display: flex;
  align-items: center;
  gap: 8px;
}

.expand-toggle {
  cursor: pointer;
  user-select: none;
  font-size: 12px;
  width: 16px;
  text-align: center;
}

.json-key {
  color: #881391;
  font-weight: bold;
}

.editable-key {
  outline: 1px dashed #bbb;
  padding: 2px 4px;
  border-radius: 3px;
  min-width: 30px;
  cursor: text;
}

.editable-key:focus {
  outline: 2px solid #3498db;
  background-color: #f8f9fa;
}

.editable-key:hover {
  outline: 1px solid #999;
  background-color: #f0f0f0;
}

.json-string {
  color: #c41a16;
}

.json-number {
  color: #1c00cf;
}

.json-boolean {
  color: #0d22aa;
}

.json-null {
  color: #808080;
}

.json-punctuation {
  color: #333;
  font-weight: bold;
}

.preview-value {
  color: #666;
  background-color: #f0f0f0;
  padding: 2px 6px;
  border-radius: 3px;
  font-style: italic;
  max-width: 300px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  margin: 0 8px;
}

.leaf-node {
  display: flex;
  align-items: center;
  gap: 4px;
  position: relative;
}

.leaf-actions {
  display: flex;
  gap: 2px;
  margin-left: 4px;
}

.editable {
  outline: 1px dashed #bbb;
  padding: 2px 4px;
  border-radius: 3px;
  max-width: 200px;
  word-break: break-all;
  white-space: pre-wrap;
  min-width: 30px;
}

.node-actions {
  display: flex;
  gap: 2px;
  margin-left: auto;
}

.action-btn {
  width: 20px;
  height: 20px;
  border: none;
  border-radius: 3px;
  cursor: pointer;
  font-size: 12px;
  font-weight: bold;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s;
}

.add-btn {
  background-color: #27ae60;
  color: white;
}

.add-btn:hover {
  background-color: #229954;
}

.add-sibling-btn {
  background-color: #3498db;
  color: white;
}

.add-sibling-btn:hover {
  background-color: #2980b9;
}

.delete-btn {
  background-color: #e74c3c;
  color: white;
}

.delete-btn:hover {
  background-color: #c0392b;
}

.leaf-delete {
  margin-left: 8px;
}

.leaf-add-sibling {
  margin-left: 2px;
}

.closing-bracket {
  margin-left: 20px;
}



@media (max-width: 768px) {
  .preview-value {
    max-width: 150px;
  }

  .editable {
    max-width: 120px;
  }
  

}
</style>




