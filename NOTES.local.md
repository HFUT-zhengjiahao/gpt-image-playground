# 本地改造说明（非上游文件）

这个仓库是在 [alasano/gpt-image-playground](https://github.com/alasano/gpt-image-playground) 基础上做的本地定制版。
本文件只记录本地改动与注意事项，方便日后维护和回退。

## 分支与回退

| 分支 / 标签 | 内容 |
| --- | --- |
| `master` / `local-base` / 标签 `before-canvas-ui` | 浅色中文版（无画布） |
| `canvas-ui` | 当前开发分支：画布视图 + 评审修复 |

```bash
git checkout local-base     # 回到无画布的浅色中文版
git checkout canvas-ui      # 回到画布版
```

桌面还有一个压缩包快照：`gpt-image-playground-备份-浅色中文版-<日期>.tar.gz`（不含 node_modules）。

## 本地改动清单

1. **服务商适配**：走 PackyAPI 中转（`.env.local` 里的 `OPENAI_API_BASE_URL=https://cf.api.fan/v1`）。
   `/api/images` 会把中转返回的图片 URL 下载后转存到本地，并支持 `response=meta`（只回 filename，不回 base64）。
2. **中文界面**：自研轻量 i18n（`src/lib/i18n/`），英文原文即 key，缺失时回退英文。
   语言选择存 cookie，`app/layout.tsx` 在服务端读取，首屏不会闪。
3. **浅色主题**：默认 light，配色以中性灰底 + 单一靛蓝强调色。
4. **画布视图**（`src/components/canvas/`）：React Flow 节点即任务，支持双击建节点、
   拖拽连线、派生编辑节点、节点内改参数重跑、血缘连线。
5. **蒙版编辑器**（`src/components/mask-editor.tsx`）：从编辑表单抽出的独立组件，画布弹窗与表单共用。
6. **关闭服务**：右上角按钮 → `POST /api/shutdown`（仅允许 localhost），另有桌面快捷方式。
7. **文件生命周期**：`POST /api/images-cleanup` 按「保留列表」清理无人引用的图片；
   历史记录删除前会检查画布引用。

## 类型检查的坑

`package.json` 里 `typescript` 被 alias 成 `npm:@typescript/typescript6`（TS 6.x），
而 `npm run typecheck` 显式走 `node_modules/typescript-7/bin/tsc`（TS 7.x）。

- **请用 `npm run typecheck`**，不要裸跑 `npx tsc`——两者解析到的版本不同。
- `npm run build`（Next 自带的类型检查）走的是 6.x。两个版本目前都能过，
  但如果将来出现「typecheck 通过、build 报类型错」的怪现象，先怀疑这里。

## 常用命令

```bash
npm run dev              # 开发服务（或双击桌面「启动 GPT Image Playground.command」）
npm run typecheck        # TS 7 类型检查
npm run lint             # ESLint（当前 0 error / 0 warning）
npm run i18n:check       # 校验每个 t('…') 都有中文词条（改文案后必跑）
npm run check            # lint + typecheck + i18n:check 三合一
npm run build            # 生产构建（已验证可过）

bash scripts/start-playground.sh   # 后台起服务 + 打开浏览器
bash scripts/stop-playground.sh    # 停止服务（等同页面右上角「关闭服务」）
```
