<template>
  <div class="page-shell">
    <header class="hero">
      <div class="hero-copy">
        <p class="eyebrow">JSON Toolkit</p>
        <h1>JSON / XML 格式化与互转工具</h1>
        <p>
          面向浏览器使用的轻量工具页，适合做 JSON 美化、压缩、校验、排序，也支持 XML 格式化和 JSON/XML 互转。
        </p>
      </div>

      <div class="hero-meta">
        <div class="meta-pill">纯前端处理</div>
        <div class="meta-pill">支持 GitHub Pages</div>
        <div class="meta-pill">JSON / XML 双格式</div>
      </div>
    </header>

    <JsonEditor />
  </div>
</template>

<script setup>
import JsonEditor from './components/JsonEditor.vue'
</script>
