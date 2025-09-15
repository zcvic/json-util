<script setup lang="ts">
import { computed } from 'vue';


const props = defineProps<{ value: unknown; path: string, expandedKeys: Record<string, boolean> }>()
const emit = defineEmits<{
  (e: 'update', newData: unknown): void,
  (e: 'toggle', key: string): void;
}>()


const isExpanded = computed(() => props.expandedKeys[props.path] ?? false)

function tmpVal(v: unknown) {
  let tmp = v
  try {
    if (typeof v === 'string')
      tmp = JSON.parse(v)
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

function onEdit(event: Event) {
  const el = event.target as HTMLElement
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
</script>

<template>
  <div class="json-item">
    <template v-if="typeof value === 'object' && value !== null">
      <span class="expand-toggle" @click="toggleExpand(path)">{{ !isExpanded ? '▶' : '▼' }}</span>
      <span class="json-punctuation">{{ Array.isArray(value) ? '[' : '{' }}</span>
      <div v-show="isExpanded">
        <div v-for="(v, k) in value" :key="k" class="json-item item">
          <span class="json-key">{{ k }}:</span>
          <JsonNode :expandedKeys="expandedKeys" :class="valColor(v)" :value="tmpVal(v)" :path="`${path}.${k}`"
            @update="updateChild(k, $event)" @toggle="toggleExpand" />
        </div>
      </div>
      <span v-show="!isExpanded">
        {{ Array.isArray(value) ? value.length : (Object.keys(value).length + 'keys') }}
      </span>
      <span class="json-punctuation">{{ Array.isArray(value) ? ']' : '}' }}</span>
    </template>
    <template v-else>
      <span class="editable" contenteditable="true" @blur="onEdit">{{ value }}</span>
    </template>
  </div>
</template>

<style scoped>
.json-item {
  display: inline-block;
  /* 减少布局影响 */
  vertical-align: top;
  margin-left: 20px;
  font-family: monospace;
  font-size: 14px;
}

.item {
  display: flex;
  margin: 15px 0;
}

.expand-toggle {
  cursor: pointer;
  margin-right: 4px;
}

.json-key {
  color: #881391;
  font-weight: bold;
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
}

.editable {
  outline: 1px dashed #bbb;
  padding: 2px 4px;
  border-radius: 3px;
  display: inline-block;
  max-width: 100%;
  word-break: break-all;
  white-space: pre-wrap;
}
</style>
