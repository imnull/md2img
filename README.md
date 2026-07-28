# md2img

Markdown → PNG 长图 / PDF 文档的纯前端 H5 工具。

## 功能

- Markdown 源码实时渲染预览
- 导出 PNG 手机长图（750px 宽）
- 导出 PDF 文档（A4 纸张，自动分页）
- 纯前端处理，无后端依赖

## 技术栈

| 技术 | 用途 |
|------|------|
| Vue 3 | 前端框架 |
| Vite | 构建工具 |
| marked | Markdown → HTML |
| html-to-image | HTML → PNG |
| jsPDF | 生成 PDF |

## 开发

```bash
pnpm install
pnpm dev
```

## 构建

```bash
pnpm build
```

产物在 `dist/` 目录，可部署到任意静态托管。
