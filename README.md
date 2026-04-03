# JSON Tool

一个基于 Vue 3 + Vite 的轻量数据格式工具页，适合在浏览器里直接完成 JSON 与 XML 的常见处理工作。

## 功能

- JSON 格式化
- JSON 压缩
- JSON 合法性校验
- 对象键按字母顺序排序
- XML 格式化
- XML 压缩
- XML 合法性校验
- JSON -> XML 互转
- XML -> JSON 互转
- 一键复制内容
- 下载转换结果文件
- 右侧结构树预览
- 字符数、行数、键数量统计

## 适合场景

- 调试接口返回数据
- 快速检查 JSON 是否合法
- 快速检查 XML 是否合法
- 美化日志或配置文件
- 对 JSON 做稳定排序后再比较差异
- 在接口调试、配置迁移或工具对接时做 JSON/XML 互转
- 在线访问一个简单好用的 JSON 处理工具

## 转换说明

- XML 转 JSON 时，会使用对象结构表示节点
- 属性会放在 `@attributes`
- 文本内容会放在 `#text`
- 重复标签会转换成数组
- JSON 转 XML 时，如果顶层只有一个键，会把该键作为根节点；否则会自动包一层 `<root>`

这个转换规则适合常见数据交换场景，但不等同于完整 XML Schema 映射。复杂命名空间、处理指令、混合内容等场景仍建议人工复核。

## 本地开发

```bash
npm install
npm run dev
```

## 生产构建

```bash
npm run build
```

## 发布到 GitHub Pages

项目已配置：

- `base: /json-tool/`
- `npm run deploy`

如果你使用 `gh-pages` 分支方式发布，可以直接执行：

```bash
npm run deploy
```

如果你使用 GitHub Actions 方式发布，也可以保留当前项目结构，只需要将 `dist/` 作为构建产物发布即可。

## 主要目录

```text
src/
  App.vue
  style.css
  components/
    JsonEditor.vue
```

## 技术栈

- Vue 3
- Vite
- Ace Editor
- vue-json-pretty
