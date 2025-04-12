<template>
  <h1 class="text-2xl p-4">🧰 JSON 工具</h1>
  <JsonEditor />
</template>

<script setup>
import JsonEditor from './components/JsonEditor.vue'
</script>
<style>
html, body, #app {
  width: 100%;
  height: 100%;
  margin: 0;
  padding: 0;
  overflow: hidden; /* ✅ 禁止页面滚动条 */
}
</style>
