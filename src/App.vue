<script setup lang="ts">
import { ref, computed, nextTick } from 'vue'
import { marked } from 'marked'
import { toPng, toCanvas } from 'html-to-image'
import jsPDF from 'jspdf'

/* ------------------------------------------------------------------ *
 * 类型
 * ------------------------------------------------------------------ */
type ExportFormat = 'png' | 'pdf'

/* ------------------------------------------------------------------ *
 * 状态
 * ------------------------------------------------------------------ */
const markdownSource = ref<string>(`# Markdown 转图片 / PDF

欢迎使用 **md2img**，一个纯前端的 Markdown 渲染与导出工具。

## 功能特性

- ✅ 实时预览 Markdown 渲染效果
- 📱 导出为手机长图（750px 宽）
- 📄 导出为 PDF 文档（A4 纸张）
- 🎨 支持代码高亮、表格、列表等常用语法

## 快速开始

1. 在左侧编辑器中输入 Markdown 源码
2. 右侧实时预览渲染效果
3. 点击下方按钮选择导出格式

\`\`\`javascript
function hello(name) {
  console.log(\`Hello, \${name}!\`)
}
hello('md2img')
\`\`\`

| 功能 | 支持状态 |
|------|:--------:|
| 实时预览 | ✅ |
| PNG 长图 | ✅ |
| PDF 文档 | ✅ |

> 纯前端处理，数据不会上传服务器，安全放心。
`)

const renderedHtml = computed<string>(() => {
  return marked.parse(markdownSource.value, { async: false }) as string
})

const isExporting = ref<boolean>(false)
const exportStatus = ref<string>('')
const previewRef = ref<HTMLElement | null>(null)

/* ------------------------------------------------------------------ *
 * 导出 PNG 长图（750px 宽）
 * ------------------------------------------------------------------ */
async function exportPng(): Promise<void> {
  if (!previewRef.value) return
  isExporting.value = true
  exportStatus.value = '正在生成 PNG 长图...'
  try {
    await nextTick()
    const dataUrl: string = await toPng(previewRef.value, {
      width: 750,
      pixelRatio: 2,
      backgroundColor: '#ffffff',
      style: {
        width: '750px',
      },
    })
    downloadDataUrl(dataUrl, 'md2img-output.png')
    exportStatus.value = 'PNG 导出成功！'
  } catch (err) {
    const msg = err instanceof Error ? err.message : String(err)
    console.error('PNG 导出失败:', err)
    exportStatus.value = '导出失败: ' + msg
  } finally {
    isExporting.value = false
    setTimeout(() => (exportStatus.value = ''), 3000)
  }
}

/* ------------------------------------------------------------------ *
 * 导出 PDF（A4）
 * ------------------------------------------------------------------ */
async function exportPdf(): Promise<void> {
  if (!previewRef.value) return
  isExporting.value = true
  exportStatus.value = '正在生成 PDF 文档...'
  try {
    await nextTick()
    // 先转 canvas
    const canvas: HTMLCanvasElement = await toCanvas(previewRef.value, {
      pixelRatio: 2,
      backgroundColor: '#ffffff',
    })

    // A4 尺寸（mm）
    const pdf = new jsPDF('p', 'mm', 'a4')
    const pdfWidth: number = pdf.internal.pageSize.getWidth()   // 210
    const pdfHeight: number = pdf.internal.pageSize.getHeight()  // 297
    const margin: number = 10
    const usableWidth: number = pdfWidth - margin * 2
    const usableHeight: number = pdfHeight - margin * 2

    // 图片在 PDF 中的高度（按比例缩放）
    const imgHeightInPdf: number = (canvas.height * usableWidth) / canvas.width

    if (imgHeightInPdf <= usableHeight) {
      // 单页
      const imgData = canvas.toDataURL('image/png')
      pdf.addImage(imgData, 'PNG', margin, margin, usableWidth, imgHeightInPdf)
    } else {
      // 多页分页
      const scale: number = canvas.width / usableWidth // px per mm
      let remainingHeight: number = imgHeightInPdf
      let offset: number = 0 // 已截取的图片高度（mm）

      while (remainingHeight > 0) {
        const pageHeight: number = Math.min(usableHeight, remainingHeight)

        // 创建该页对应的 canvas 片段
        const pageCanvas: HTMLCanvasElement = document.createElement('canvas')
        pageCanvas.width = canvas.width
        pageCanvas.height = Math.ceil(pageHeight * scale)
        const ctx: CanvasRenderingContext2D | null = pageCanvas.getContext('2d')
        if (!ctx) throw new Error('无法获取 canvas 2D context')

        ctx.fillStyle = '#ffffff'
        ctx.fillRect(0, 0, pageCanvas.width, pageCanvas.height)
        ctx.drawImage(
          canvas,
          0, offset * scale,
          canvas.width, pageCanvas.height * scale,
          0, 0,
          canvas.width, pageCanvas.height * scale
        )

        const pageImgData = pageCanvas.toDataURL('image/png')
        pdf.addImage(pageImgData, 'PNG', margin, margin, usableWidth, pageHeight)

        remainingHeight -= usableHeight
        offset += usableHeight
        if (remainingHeight > 0) {
          pdf.addPage()
        }
      }
    }

    pdf.save('md2img-output.pdf')
    exportStatus.value = 'PDF 导出成功！'
  } catch (err) {
    const msg = err instanceof Error ? err.message : String(err)
    console.error('PDF 导出失败:', err)
    exportStatus.value = '导出失败: ' + msg
  } finally {
    isExporting.value = false
    setTimeout(() => (exportStatus.value = ''), 3000)
  }
}

/* ------------------------------------------------------------------ *
 * 工具函数
 * ------------------------------------------------------------------ */
function downloadDataUrl(dataUrl: string, filename: string): void {
  const link: HTMLAnchorElement = document.createElement('a')
  link.href = dataUrl
  link.download = filename
  document.body.appendChild(link)
  link.click()
  document.body.removeChild(link)
}

function insertSample(): void {
  markdownSource.value = `# 示例文档

## 1. 文本样式

**加粗文本** *斜体文本* ~~删除线~~ \`行内代码\`

## 2. 列表

### 无序列表
- 项目 A
- 项目 B
- 项目 C

### 有序列表
1. 第一步
2. 第二步
3. 第三步

## 3. 代码块

\`\`\`python
def fibonacci(n):
    a, b = 0, 1
    for _ in range(n):
        print(a, end=' ')
        a, b = b, a + b
fibonacci(10)
\`\`\`

## 4. 表格

| 指标 | 数值 | 说明 |
|------|-----:|------|
| CPU | 3.2GHz | 主频 |
| 内存 | 16GB | DDR5 |
| 存储 | 1TB | NVMe |

## 5. 引用

> 这是一个引用块。
>
> 可以包含多行内容。

## 6. 分割线

---

以上是示例内容，你可以清空编辑器输入自己的 Markdown。
`
}
</script>

<template>
  <div class="app">
    <!-- 顶部栏 -->
    <header class="app-header">
      <h1 class="app-title">
        <span class="logo">md2img</span>
        <span class="subtitle">Markdown → PNG / PDF</span>
      </h1>
    </header>

    <!-- 主体：左编辑 右预览 -->
    <main class="app-main">
      <!-- 左：编辑器 -->
      <section class="editor-panel">
        <div class="panel-header">
          <span class="panel-label">Markdown 源码</span>
          <div class="panel-actions">
            <button class="btn-text" @click="insertSample" title="插入示例">插入示例</button>
            <button class="btn-text" @click="markdownSource = ''" title="清空">清空</button>
          </div>
        </div>
        <textarea
          v-model="markdownSource"
          class="editor-textarea"
          spellcheck="false"
          placeholder="在此输入 Markdown 源码..."
        ></textarea>
      </section>

      <!-- 右：预览 -->
      <section class="preview-panel">
        <div class="panel-header">
          <span class="panel-label">渲染预览</span>
          <span class="export-hint">导出宽度: 750px</span>
        </div>
        <div class="preview-scroll">
          <!-- 预览容器：宽度固定 750px，与导出尺寸一致 -->
          <div class="preview-wrapper">
            <div ref="previewRef" class="markdown-body" v-html="renderedHtml"></div>
          </div>
        </div>
      </section>
    </main>

    <!-- 底部：导出栏 -->
    <footer class="app-footer">
      <div class="export-buttons">
        <button
          class="btn-export btn-png"
          :disabled="isExporting"
          @click="exportPng"
        >
          <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <rect x="3" y="3" width="18" height="18" rx="2"/>
            <circle cx="8.5" cy="8.5" r="1.5"/>
            <path d="M21 15l-5-5L5 21"/>
          </svg>
          导出 PNG 长图
        </button>
        <button
          class="btn-export btn-pdf"
          :disabled="isExporting"
          @click="exportPdf"
        >
          <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/>
            <path d="M14 2v6h6"/>
            <path d="M9 13h6M9 17h6"/>
          </svg>
          导出 PDF (A4)
        </button>
      </div>
      <div class="export-status" :class="{ active: exportStatus }">
        {{ exportStatus }}
      </div>
    </footer>
  </div>
</template>
