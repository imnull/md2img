# md2img

Markdown → PNG 长图 / PDF 文档的纯前端 H5 工具。

线上：<https://md2img.mkjs.net/>

## 功能

- Markdown 源码实时渲染预览
- 导出 PNG 手机长图（750px 宽）
- 导出 PDF 文档（A4 纸张，自动分页）
- 导出物底部自动附二维码 + 项目名 MD2IMG
  - PNG：二维码与文字在长图底部（DOM 渲染，随预览 WYSIWYG）
  - PDF：每页右下角 3mm 边距处盖 `[QR] MD2IMG` 水印（jsPDF 直接绘制）
- 纯前端处理，无后端依赖

## 技术栈

| 技术 | 用途 |
|------|------|
| Vue 3 (`<script setup lang="ts">`) | 前端框架 |
| Vite | 构建工具 |
| marked | Markdown → HTML |
| html-to-image | HTML → PNG / Canvas |
| jsPDF | 生成 PDF |
| qrcode | 生成 https://md2img.mkjs.net/ 的二维码 dataURL |

## 开发

```bash
pnpm install
pnpm dev
```

## 构建

```bash
pnpm build
```

产物在 `dist/` 目录。`base: '/'`（独立域名根路径，见 `vite.config.ts`）。

## 界面

- 左侧：Markdown 源码编辑器（带「插入示例」「清空」按钮）
- 右侧：实时渲染预览（固定 750px 宽，与导出尺寸一致，WYSIWYG）
- 底部：两个导出按钮 —— `导出PNG`、`导出PDF`

## 部署

push 到 `main` 分支即触发 GitHub Actions 自动部署。

- 工作流文件：`.github/workflows/deploy.yml`
- 流程：`pnpm build` → `rsync -avz --delete dist/ root@md2img.mkjs.net:/var/www/md2img/` → curl 验证 HTTPS 200
- 部署密钥：本机 `~/.ssh/md2img-deploy/id_ed25519`（ed25519，无密码），公钥已加入服务器 `/root/.ssh/authorized_keys`

### 所需 GitHub Secrets

仓库 Settings → Secrets and variables → Actions：

| 名称 | 值 |
|------|----|
| `SSH_PRIVATE_KEY` | 部署私钥（与服务器 authorized_keys 中的公钥配对） |
| `SSH_HOST` | `md2img.mkjs.net` |

## 服务器基础设施

- 主机：Aliyun ECS（Ubuntu 22.04），用户 `root` 可从本机免密 SSH
- nginx：配置 `/etc/nginx/conf.d/md2img.mkjs.net.conf`，web root `/var/www/md2img`
- SSL：Let's Encrypt，certbot `--nginx` 自动签发并续期，证书路径 `/etc/letsencrypt/live/md2img.mkjs.net/`
- 同一服务器还托管 `zenimg.mkjs.net`（同模板）
