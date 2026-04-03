<template>
  <section class="tool-card">
    <div class="toolbar">
      <div class="toolbar-left">
        <label class="field">
          <span>源格式</span>
          <select v-model="sourceFormat">
            <option value="json">JSON</option>
            <option value="xml">XML</option>
          </select>
        </label>

        <label class="field">
          <span>目标格式</span>
          <select v-model="targetFormat">
            <option value="json">JSON</option>
            <option value="xml">XML</option>
          </select>
        </label>

        <label class="field">
          <span>缩进</span>
          <select v-model.number="indentSize">
            <option :value="2">2 空格</option>
            <option :value="4">4 空格</option>
          </select>
        </label>

        <div class="stats">
          <span>{{ stats.characters }} 字符</span>
          <span>{{ stats.lines }} 行</span>
          <span>{{ stats.keys }} 个键</span>
        </div>
      </div>

      <div class="toolbar-actions">
        <button class="primary-btn" @click="formatInput">格式化</button>
        <button class="secondary-btn" @click="minifyInput">压缩</button>
        <button class="secondary-btn" @click="validateInput">校验</button>
        <button class="secondary-btn" @click="convertContent">互转</button>
        <button class="secondary-btn" @click="copyResult">复制结果</button>
        <button class="secondary-btn" @click="downloadResult">下载结果</button>
        <button class="secondary-btn" @click="loadExample">示例</button>
        <button class="secondary-btn" @click="clearAll">清空</button>
      </div>
    </div>

    <div class="workspace">
      <section class="panel editor-panel">
        <div class="panel-header">
          <div>
            <h2>输入区</h2>
            <p>当前支持 JSON 编辑、XML 编辑，以及两者之间的基础互转。</p>
          </div>

          <span class="status-pill" :class="statusClass">
            {{ statusLabel }}
          </span>
        </div>

        <div class="editor-container" ref="editorRef"></div>
      </section>

      <section class="panel result-panel">
        <div class="panel-header">
          <div>
            <h2>转换结果</h2>
            <p>{{ resultHint }}</p>
          </div>
        </div>

        <textarea
          v-model="resultContent"
          class="result-output"
          readonly
          spellcheck="false"
          placeholder="点击“互转”后，在这里查看转换结果"
        />

        <div class="preview-wrapper">
          <div class="preview-header">
            <h3>结构预览</h3>
            <span v-if="previewFormat === 'json' && isValid">以 JSON 结构树显示</span>
            <span v-else>仅在源内容可解析时显示</span>
          </div>

          <div v-if="isValid" class="viewer-container">
            <vue-json-pretty
              :data="previewJson"
              :deep="3"
              :show-line-number="false"
              :show-length="true"
              :show-double-quotes="true"
            />
          </div>

          <div v-else class="empty-state">
            <strong>当前内容无法解析</strong>
            <p>{{ error || '请输入合法 JSON 或 XML 后再查看结构。' }}</p>
          </div>
        </div>
      </section>
    </div>

    <div class="status-bar" :class="statusClass">
      <span>{{ statusMessage }}</span>
    </div>
  </section>
</template>

<script setup>
import { computed, onBeforeUnmount, onMounted, reactive, ref, watch } from 'vue'
import ace from 'ace-builds/src-noconflict/ace'
import 'ace-builds/src-noconflict/ext-language_tools'
import 'ace-builds/src-noconflict/mode-json'
import 'ace-builds/src-noconflict/mode-xml'
import 'ace-builds/src-noconflict/theme-github'
import VueJsonPretty from 'vue-json-pretty'
import 'vue-json-pretty/lib/styles.css'

const JSON_EXAMPLE = `{
  "project": "json-tool",
  "enabled": true,
  "features": [
    "beautify",
    "minify",
    "validate",
    "json-xml-convert"
  ],
  "meta": {
    "author": "Chason-gyc",
    "version": 3
  }
}`

const XML_EXAMPLE = `<?xml version="1.0" encoding="UTF-8"?>
<project enabled="true">
  <name>json-tool</name>
  <features>
    <feature>beautify</feature>
    <feature>minify</feature>
    <feature>validate</feature>
    <feature>json-xml-convert</feature>
  </features>
  <meta>
    <author>Chason-gyc</author>
    <version>3</version>
  </meta>
</project>`

const editorRef = ref(null)
const sourceFormat = ref('json')
const targetFormat = ref('xml')
const indentSize = ref(2)
const resultContent = ref('')
const previewJson = ref({})
const parseState = ref('idle')
const message = ref('等待输入 JSON 或 XML 内容')
const error = ref('')
const stats = reactive({
  characters: 0,
  lines: 0,
  keys: 0,
})

let editor

const isValid = computed(() => parseState.value === 'success')

const statusClass = computed(() => {
  if (parseState.value === 'success') {
    return 'success'
  }
  if (parseState.value === 'error') {
    return 'error'
  }
  return 'idle'
})

const statusLabel = computed(() => {
  if (parseState.value === 'success') {
    return '解析成功'
  }
  if (parseState.value === 'error') {
    return '解析失败'
  }
  return '待校验'
})

const statusMessage = computed(() => error.value || message.value)

const resultHint = computed(() => {
  if (sourceFormat.value === targetFormat.value) {
    return `当前源格式与目标格式相同，互转时会输出规范化后的 ${targetFormat.value.toUpperCase()}。`
  }

  return `${sourceFormat.value.toUpperCase()} -> ${targetFormat.value.toUpperCase()}`
})

const previewFormat = computed(() => 'json')

const countKeys = (value) => {
  if (Array.isArray(value)) {
    return value.reduce((sum, item) => sum + countKeys(item), 0)
  }

  if (value && typeof value === 'object') {
    return Object.keys(value).length + Object.values(value).reduce((sum, item) => sum + countKeys(item), 0)
  }

  return 0
}

const updateStats = (content, parsedValue = null) => {
  stats.characters = content.length
  stats.lines = content ? content.split(/\r?\n/).length : 0
  stats.keys = parsedValue ? countKeys(parsedValue) : 0
}

const xmlEscape = (value) =>
  String(value)
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&apos;')

const formatXmlString = (xmlString, indentSizeValue = 2) => {
  const indent = ' '.repeat(indentSizeValue)
  const normalized = xmlString.replace(/>\s*</g, '><').trim()
  const tokens = normalized.replace(/(>)(<)(\/*)/g, '$1\n$2$3').split('\n')
  let level = 0

  return tokens
    .map((token) => {
      if (/^<\//.test(token)) {
        level -= 1
      }

      const line = `${indent.repeat(Math.max(level, 0))}${token}`

      if (
        /^<[^!?/][^>]*[^/]>\s*$/.test(token) &&
        !/<\/[^>]+>$/.test(token) &&
        !/^<[^>]+>.*<\/[^>]+>$/.test(token)
      ) {
        level += 1
      }

      return line
    })
    .join('\n')
}

const minifyXmlString = (xmlString) => xmlString.replace(/>\s+</g, '><').trim()

const setEditorMode = () => {
  const mode = sourceFormat.value === 'json' ? 'ace/mode/json' : 'ace/mode/xml'
  editor.session.setMode(mode)
}

const setEditorValue = (value) => {
  editor.setValue(value, 1)
  editor.clearSelection()
}

const parseXmlDocument = (content) => {
  const documentNode = new DOMParser().parseFromString(content, 'application/xml')
  const parserError = documentNode.querySelector('parsererror')

  if (parserError) {
    throw new Error(parserError.textContent.replace(/\s+/g, ' ').trim())
  }

  return documentNode
}

const xmlElementToJson = (element) => {
  const result = {}
  const attributes = [...element.attributes]
  const childElements = [...element.children]
  const textNodes = [...element.childNodes]
    .filter((node) => node.nodeType === Node.TEXT_NODE)
    .map((node) => node.nodeValue.trim())
    .filter(Boolean)

  if (attributes.length) {
    result['@attributes'] = attributes.reduce((bucket, attribute) => {
      bucket[attribute.name] = attribute.value
      return bucket
    }, {})
  }

  if (!childElements.length) {
    if (!attributes.length && textNodes.length <= 1) {
      return textNodes[0] ?? ''
    }

    if (textNodes.length) {
      result['#text'] = textNodes.join(' ')
    }

    return result
  }

  childElements.forEach((child) => {
    const value = xmlElementToJson(child)
    if (child.tagName in result) {
      if (!Array.isArray(result[child.tagName])) {
        result[child.tagName] = [result[child.tagName]]
      }
      result[child.tagName].push(value)
    } else {
      result[child.tagName] = value
    }
  })

  if (textNodes.length) {
    result['#text'] = textNodes.join(' ')
  }

  return result
}

const xmlToJsonObject = (documentNode) => ({
  [documentNode.documentElement.tagName]: xmlElementToJson(documentNode.documentElement),
})

const objectToXmlNode = (tagName, value, level = 0) => {
  const indent = ' '.repeat(indentSize.value)
  const prefix = indent.repeat(level)

  if (Array.isArray(value)) {
    return value.map((item) => objectToXmlNode(tagName, item, level)).join('\n')
  }

  if (value === null || value === undefined) {
    return `${prefix}<${tagName}></${tagName}>`
  }

  if (typeof value !== 'object') {
    return `${prefix}<${tagName}>${xmlEscape(value)}</${tagName}>`
  }

  const attributes = value['@attributes'] || {}
  const textValue = value['#text']
  const attrText = Object.entries(attributes)
    .map(([key, attrValue]) => ` ${key}="${xmlEscape(attrValue)}"`)
    .join('')

  const childEntries = Object.entries(value).filter(([key]) => key !== '@attributes' && key !== '#text')

  if (!childEntries.length && textValue === undefined) {
    return `${prefix}<${tagName}${attrText}></${tagName}>`
  }

  if (!childEntries.length && textValue !== undefined) {
    return `${prefix}<${tagName}${attrText}>${xmlEscape(textValue)}</${tagName}>`
  }

  const childContent = childEntries
    .flatMap(([childTag, childValue]) => {
      if (Array.isArray(childValue)) {
        return childValue.map((item) => objectToXmlNode(childTag, item, level + 1))
      }
      return [objectToXmlNode(childTag, childValue, level + 1)]
    })
    .join('\n')

  const textLine = textValue !== undefined ? `\n${indent.repeat(level + 1)}${xmlEscape(textValue)}` : ''

  return `${prefix}<${tagName}${attrText}>${textLine}${childContent ? `\n${childContent}` : ''}\n${prefix}</${tagName}>`
}

const jsonObjectToXml = (value) => {
  let rootTag = 'root'
  let rootValue = value

  if (
    value &&
    typeof value === 'object' &&
    !Array.isArray(value) &&
    Object.keys(value).length === 1
  ) {
    rootTag = Object.keys(value)[0]
    rootValue = value[rootTag]
  }

  const body = objectToXmlNode(rootTag, rootValue, 0)
  return `<?xml version="1.0" encoding="UTF-8"?>\n${body}`
}

const parseInputContent = () => {
  const content = editor.getValue()

  if (!content.trim()) {
    previewJson.value = {}
    parseState.value = 'idle'
    message.value = '等待输入 JSON 或 XML 内容'
    error.value = ''
    updateStats(content)
    return null
  }

  try {
    if (sourceFormat.value === 'json') {
      const parsedValue = JSON.parse(content)
      previewJson.value = parsedValue
      parseState.value = 'success'
      message.value = 'JSON 解析成功'
      error.value = ''
      updateStats(content, parsedValue)
      return {
        source: 'json',
        raw: parsedValue,
        preview: parsedValue,
      }
    }

    const documentNode = parseXmlDocument(content)
    const jsonValue = xmlToJsonObject(documentNode)
    previewJson.value = jsonValue
    parseState.value = 'success'
    message.value = 'XML 解析成功'
    error.value = ''
    updateStats(content, jsonValue)
    return {
      source: 'xml',
      raw: documentNode,
      preview: jsonValue,
    }
  } catch (parseError) {
    previewJson.value = {}
    parseState.value = 'error'
    message.value = ''
    error.value = parseError.message
    updateStats(content)
    return null
  }
}

const resizeEditor = () => {
  if (editor) {
    editor.resize()
  }
}

const formatInput = () => {
  const parsedState = parseInputContent()
  if (!parsedState) {
    return
  }

  if (sourceFormat.value === 'json') {
    setEditorValue(JSON.stringify(parsedState.raw, null, indentSize.value))
    resultContent.value = ''
    message.value = 'JSON 已格式化'
    return
  }

  const serialized = new XMLSerializer().serializeToString(parsedState.raw)
  setEditorValue(formatXmlString(serialized, indentSize.value))
  resultContent.value = ''
  message.value = 'XML 已格式化'
}

const minifyInput = () => {
  const parsedState = parseInputContent()
  if (!parsedState) {
    return
  }

  if (sourceFormat.value === 'json') {
    setEditorValue(JSON.stringify(parsedState.raw))
    resultContent.value = ''
    message.value = 'JSON 已压缩'
    return
  }

  const serialized = new XMLSerializer().serializeToString(parsedState.raw)
  setEditorValue(minifyXmlString(serialized))
  resultContent.value = ''
  message.value = 'XML 已压缩'
}

const validateInput = () => {
  const parsedState = parseInputContent()
  if (!parsedState) {
    return
  }

  message.value = `${sourceFormat.value.toUpperCase()} 校验通过`
}

const convertContent = () => {
  const parsedState = parseInputContent()
  if (!parsedState) {
    return
  }

  if (targetFormat.value === 'json') {
    const jsonOutput = sourceFormat.value === 'json' ? parsedState.raw : parsedState.preview
    resultContent.value = JSON.stringify(jsonOutput, null, indentSize.value)
    message.value = `${sourceFormat.value.toUpperCase()} -> JSON 转换完成`
    return
  }

  const sourceValue = sourceFormat.value === 'json' ? parsedState.raw : parsedState.preview
  resultContent.value = jsonObjectToXml(sourceValue)
  message.value = `${sourceFormat.value.toUpperCase()} -> XML 转换完成`
}

const copyResult = async () => {
  if (!resultContent.value.trim()) {
    parseState.value = 'error'
    error.value = '当前没有可复制的转换结果'
    message.value = ''
    return
  }

  try {
    await navigator.clipboard.writeText(resultContent.value)
    parseState.value = 'success'
    error.value = ''
    message.value = '转换结果已复制到剪贴板'
  } catch {
    parseState.value = 'error'
    error.value = '浏览器未允许访问剪贴板，请手动复制'
    message.value = ''
  }
}

const downloadResult = () => {
  if (!resultContent.value.trim()) {
    parseState.value = 'error'
    error.value = '请先执行互转后再下载结果'
    message.value = ''
    return
  }

  const extension = targetFormat.value === 'json' ? 'json' : 'xml'
  const blob = new Blob([resultContent.value], {
    type: targetFormat.value === 'json' ? 'application/json;charset=utf-8' : 'application/xml;charset=utf-8',
  })
  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')
  link.href = url
  link.download = `converted.${extension}`
  link.click()
  URL.revokeObjectURL(url)
  message.value = '转换结果已开始下载'
  parseState.value = 'success'
}

const loadExample = () => {
  const value = sourceFormat.value === 'json' ? JSON_EXAMPLE : XML_EXAMPLE
  setEditorValue(value)
  resultContent.value = ''
  parseInputContent()
  message.value = `已载入 ${sourceFormat.value.toUpperCase()} 示例`
}

const clearAll = () => {
  setEditorValue('')
  resultContent.value = ''
  parseInputContent()
}

watch(sourceFormat, (nextFormat) => {
  targetFormat.value = nextFormat === 'json' ? 'xml' : 'json'

  if (editor) {
    setEditorMode()
    if (!editor.getValue().trim()) {
      loadExample()
    } else {
      parseInputContent()
    }
  }
})

watch(indentSize, (nextValue) => {
  if (editor) {
    editor.session.setTabSize(nextValue)
  }
})

onMounted(() => {
  editor = ace.edit(editorRef.value)
  editor.setTheme('ace/theme/github')
  editor.setOptions({
    fontSize: 14,
    showPrintMargin: false,
    useWorker: true,
    tabSize: indentSize.value,
    wrap: true,
    enableBasicAutocompletion: true,
    enableLiveAutocompletion: true,
  })

  setEditorMode()
  setEditorValue(JSON_EXAMPLE)
  parseInputContent()

  editor.session.on('change', () => {
    parseInputContent()
  })

  window.addEventListener('resize', resizeEditor)
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', resizeEditor)
  if (editor) {
    editor.destroy()
    editor.container.remove()
  }
})
</script>

<style scoped>
.tool-card {
  margin-top: 22px;
  border-radius: 26px;
  border: 1px solid rgba(15, 23, 42, 0.08);
  background: rgba(255, 255, 255, 0.88);
  box-shadow: 0 16px 44px rgba(15, 23, 42, 0.08);
  overflow: hidden;
}

.toolbar {
  display: flex;
  justify-content: space-between;
  gap: 20px;
  padding: 20px 22px;
  border-bottom: 1px solid rgba(15, 23, 42, 0.08);
  background: rgba(248, 250, 252, 0.92);
}

.toolbar-left {
  display: flex;
  align-items: end;
  gap: 18px;
  flex-wrap: wrap;
}

.field {
  display: flex;
  flex-direction: column;
  gap: 8px;
  font-size: 14px;
  font-weight: 600;
}

.field select {
  min-height: 42px;
  min-width: 112px;
  padding: 0 12px;
  border: 1px solid #d4dce7;
  border-radius: 12px;
  background: #fff;
}

.stats {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  color: #52627a;
  font-size: 14px;
}

.stats span {
  padding: 8px 12px;
  border-radius: 999px;
  background: #eef2f7;
}

.toolbar-actions {
  display: flex;
  flex-wrap: wrap;
  justify-content: flex-end;
  gap: 10px;
}

.primary-btn,
.secondary-btn {
  min-height: 42px;
  padding: 0 16px;
  border: none;
  border-radius: 12px;
  cursor: pointer;
  transition:
    transform 0.18s ease,
    box-shadow 0.18s ease,
    background-color 0.18s ease;
}

.primary-btn {
  background: linear-gradient(135deg, #0f766e, #14b8a6);
  color: #fff;
  box-shadow: 0 10px 24px rgba(20, 184, 166, 0.2);
}

.secondary-btn {
  background: #e7edf5;
  color: #14213d;
}

.primary-btn:hover,
.secondary-btn:hover {
  transform: translateY(-1px);
}

.workspace {
  display: grid;
  grid-template-columns: minmax(0, 1.1fr) minmax(360px, 0.9fr);
  min-height: 680px;
}

.panel {
  display: flex;
  flex-direction: column;
  min-height: 680px;
}

.editor-panel {
  border-right: 1px solid rgba(15, 23, 42, 0.08);
}

.panel-header {
  display: flex;
  justify-content: space-between;
  align-items: start;
  gap: 12px;
  padding: 18px 20px 14px;
}

.panel-header h2,
.preview-header h3 {
  margin: 0;
  font-size: 1.05rem;
}

.panel-header p,
.preview-header span {
  margin: 6px 0 0;
  color: #52627a;
  font-size: 0.92rem;
}

.status-pill {
  flex-shrink: 0;
  padding: 8px 12px;
  border-radius: 999px;
  font-size: 13px;
  font-weight: 700;
}

.status-pill.success {
  background: #dcfce7;
  color: #166534;
}

.status-pill.error {
  background: #fee2e2;
  color: #991b1b;
}

.status-pill.idle {
  background: #e2e8f0;
  color: #334155;
}

.editor-container {
  flex: 1;
  min-height: 580px;
}

.result-output {
  min-height: 220px;
  margin: 0 20px;
  padding: 16px;
  border: 1px solid #d7dee8;
  border-radius: 18px;
  background: #f8fafc;
  color: #0f172a;
  resize: vertical;
  outline: none;
}

.preview-wrapper {
  display: flex;
  flex-direction: column;
  flex: 1;
  min-height: 0;
  padding: 18px 20px 20px;
}

.preview-header {
  display: flex;
  justify-content: space-between;
  gap: 12px;
  align-items: start;
  margin-bottom: 14px;
}

.viewer-container {
  flex: 1;
  min-height: 240px;
  overflow: auto;
}

.empty-state {
  padding: 18px;
  border-radius: 18px;
  background: #f8fafc;
  color: #475569;
}

.empty-state strong {
  display: block;
  margin-bottom: 8px;
  color: #0f172a;
}

.status-bar {
  padding: 12px 18px;
  border-top: 1px solid rgba(15, 23, 42, 0.08);
  font-size: 14px;
}

.status-bar.success {
  color: #166534;
  background: rgba(220, 252, 231, 0.7);
}

.status-bar.error {
  color: #991b1b;
  background: rgba(254, 226, 226, 0.75);
}

.status-bar.idle {
  color: #334155;
  background: rgba(241, 245, 249, 0.9);
}

@media (max-width: 1160px) {
  .toolbar {
    flex-direction: column;
  }

  .workspace {
    grid-template-columns: 1fr;
  }

  .panel,
  .editor-panel {
    min-height: 520px;
    border-right: none;
  }

  .editor-panel {
    border-bottom: 1px solid rgba(15, 23, 42, 0.08);
  }
}
</style>
