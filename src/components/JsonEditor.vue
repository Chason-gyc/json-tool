<template>
  <div class="json-page">
    <!-- 顶部按钮 -->
    <div class="top-bar">
      <button @click="beautify" class="btn">美化</button>
      <button @click="minify" class="btn">压缩</button>
      <button @click="validate" class="btn">校验</button>
      <button @click="copy" class="btn">复制</button>
    </div>

    <!-- 主体区域：左右布局 -->
    <div class="main-content">
      <!-- 左侧编辑器 -->
      <div class="editor-container" ref="editorRef"></div>

      <!-- 右侧 JSON 树 -->
      <div class="viewer-container">
        <vue-json-pretty :data="parsedJson" :deep="2" :showLineNumber="false" />
      </div>
    </div>

    <!-- 状态栏 -->
    <div class="status-bar" :class="{ ok: message, err: error }">
      <span v-if="message">{{ message }}</span>
      <span v-if="error">{{ error }}</span>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, watch } from 'vue'
import ace from 'ace-builds/src-noconflict/ace'
import 'ace-builds/src-noconflict/mode-json'
import 'ace-builds/src-noconflict/theme-github'
import VueJsonPretty from 'vue-json-pretty'
import 'vue-json-pretty/lib/styles.css'

const editorRef = ref(null)
let editor

const message = ref('')
const error = ref('')
const parsedJson = ref({}) // 用于右侧展示的 JSON

const updateTree = () => {
  try {
    parsedJson.value = JSON.parse(editor.getValue())
  } catch {
    parsedJson.value = {}
  }
}

const resizeEditor = () => {
  if (editor) editor.resize()
}

onMounted(() => {
  editor = ace.edit(editorRef.value)
  editor.session.setMode('ace/mode/json')
  editor.setTheme('ace/theme/github')
  editor.setValue('{\n  "hello": "world",\n  "list": [1, 2, 3]\n}', 1)
  editor.setOptions({ maxLines: Infinity })

  editor.session.on('change', updateTree)
  updateTree()

  window.addEventListener('resize', resizeEditor)
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', resizeEditor)
})

const beautify = () => {
  try {
    const json = JSON.parse(editor.getValue())
    editor.setValue(JSON.stringify(json, null, 2), 1)
    message.value = '格式化成功 ✅'
    error.value = ''
  } catch {
    error.value = 'JSON格式错误 ❌'
    message.value = ''
  }
}

const minify = () => {
  try {
    const json = JSON.parse(editor.getValue())
    editor.setValue(JSON.stringify(json), 1)
    message.value = '压缩成功 ✅'
    error.value = ''
  } catch {
    error.value = 'JSON格式错误 ❌'
    message.value = ''
  }
}

const validate = () => {
  try {
    JSON.parse(editor.getValue())
    message.value = 'JSON 格式正确 ✅'
    error.value = ''
  } catch {
    error.value = '格式错误 ❌'
    message.value = ''
  }
}

const copy = async () => {
  try {
    await navigator.clipboard.writeText(editor.getValue())
    message.value = '复制成功 ✅'
    error.value = ''
  } catch {
    error.value = '复制失败 ❌'
    message.value = ''
  }
}
</script>

<style scoped>
.json-page {
  display: flex;
  flex-direction: column;
  height: 100vh;
  margin: 0;
  overflow: hidden;
  font-family: "Segoe UI", Roboto, sans-serif;
}

.top-bar {
  height: 48px;
  display: flex;
  align-items: center;
  padding: 0 16px;
  gap: 12px;
  background: #f1f5f9;
  border-bottom: 1px solid #d1d5db;
  flex-shrink: 0;
}

.main-content {
  flex: 1;
  display: flex;
  width: 100%;
  overflow: hidden;
}

.editor-container,
.viewer-container {
  flex: 1;
  height: 100%;
  overflow: auto;
  border-right: 1px solid #eee;
}

.viewer-container {
  padding: 12px;
  background-color: #f9fafb;
}

.status-bar {
  height: 36px;
  padding: 6px 16px;
  font-size: 14px;
  color: #444;
  background-color: #f9fafb;
  border-top: 1px solid #e5e7eb;
  flex-shrink: 0;
}
.status-bar.ok {
  color: #16a34a;
}
.status-bar.err {
  color: #dc2626;
}

.btn {
  padding: 6px 14px;
  background-color: #2563eb;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-size: 14px;
}
.btn:hover {
  background-color: #1d4ed8;
}
</style>
