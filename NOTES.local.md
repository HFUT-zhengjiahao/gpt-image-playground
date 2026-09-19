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

## 项目位置与启动图标

- 项目已从 `~/Desktop/gpt-image-playground` 移到 **`~/gpt-image-playground`**。
  原因：macOS 把「桌面」列为受保护目录（TCC），未签名的 .app 没有磁盘访问授权，
  双击图标时会以 `Operation not permitted` 失败。家目录根不受保护，图标即可正常工作。
- **app 本体在 `~/Applications/GPT Image Playground.app`**（自建 bundle，非签名应用）。
  双击 = 启动服务并打开浏览器；已在运行时只打开浏览器，不重启服务（可反复双击）。
  也可以把它拖到程序坞。
- 桌面上的 `GPT Image Playground` 是指向该 app 的 **Finder 别名**。
  ⚠️ 必须是别名，**不要改成软链接**：软链接会让 macOS 把 app 判定为位于桌面（受 TCC 保护），
  于是读项目文件时报 `Operation not permitted` —— 这正是历史上图标启动失败的原因。
- app 内部结构：`Contents/MacOS/launcher`（bash 脚本，唯一逻辑）+ `Contents/Info.plist`
  + `Contents/Resources/AppIcon.icns`。launcher 默认把项目定位到 `$HOME/gpt-image-playground`；
  如果 .app 被放回项目根，则自动改用 .app 所在位置。
- 图标源文件是 SF Symbol `photo.on.rectangle.angled` + 靛蓝渐变，由 `/tmp/make-icon.swift`
  渲染（如已删除，按同样思路用 `swiftc` + AppKit 重画即可）。
- 异常记录在 `~/Library/Logs/gpt-image-playground-launcher.log`；启动失败会弹警告框。
- 停止服务不需要单独图标：画布左下角 **设置 → 服务 → 关闭服务**，
  或命令行 `bash scripts/stop-playground.sh`。
- 启动脚本 `scripts/start-playground.sh` 必须保持**可读**（`chmod 755`）。
  一旦被改成 `711` 之类没有读权限的模式，启动器会报 `Operation not permitted`。

### 排查图标启动失败

```bash
tail -30 ~/Library/Logs/gpt-image-playground-launcher.log   # 第一现场
tail -30 .run/dev.log                                       # Next.js 自己的输出
bash scripts/start-playground.sh                            # 绕过 app 直接验证脚本能否启动
```

## 安全与运维注意点

- **鉴权**：`src/lib/api-auth.ts` 的 `checkPassword()` 是唯一的密码校验入口（images / image-upload /
  image-delete / images-cleanup / settings / shutdown 六处）。未设 `APP_PASSWORD` 时全部放行（本地自用默认）。
- **登记表白名单**：`index.json` 丢失后重建时，只登记符合 `\d{13}-\d+.<ext>` 与 `upload-*.<ext>`
  的文件（`isOwnedFilename`）。把输出目录指向装满私人图片的文件夹时，那些文件永远不会被清理。
- **回收站**：删除=移动到 `<输出目录>/.trash/<日期>/`，保留天数在设置页配置（1–3650，默认 30）。
- **局域网访问**：dev server 仍监听所有网卡，方便手机/平板访问。若要收紧，把
  `scripts/start-playground.sh` 里的 `next dev` 改成 `next dev -H 127.0.0.1`——代价是局域网设备无法访问。
  `/api/shutdown` 已收紧 Host 精确匹配 + 密码校验。
- **图片缓存**：`/api/image/<文件名>` 返回 `Cache-Control: immutable`。文件名含创建时间戳且从不原地修改，
  所以改图必须换文件名，否则浏览器会一直用旧图。

## 界面结构（画布 / 历史 / 设置）

列表式表单界面已删除，现在是三件套：

- **画布**（默认视图）：左侧画布列表 + React Flow 工作台；节点=一次生成/编辑任务
- **历史记录**：全屏画廊，搜索提示词、按日期分组、详情弹窗（大图/提示词/参数/费用/下载/发送到画布）
- **设置**（画布左下角齿轮）：图片保存目录、新建节点默认模型/质量、访问密码

相关接口：

| 接口 | 用途 |
| --- | --- |
| `GET/PUT /api/settings` | 读写 `outputDir`（可带 `moveExisting` 迁移已有图片）与回收站保留天数 |
| `GET /api/images-list` | 列出输出目录里的全部图片（用于按磁盘重建历史） |
| `POST /api/images-cleanup` | 清理无人引用的图片（移入回收站，保留 30 天） |

设置存放在项目根的 `.playground-settings.json`；画布只记录文件名，所以换目录不影响任何画布。

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

## 参考图数量（多图编辑）

模型本身支持 **16 张**参考图（OpenAI `images.edit` 文档），限制通常来自中转自己的校验层。

实测（2026-09-18）：PackyAPI 文档写「建议一次只上传 1 张」，但实际 **2 张、4 张都能正常转发**，
所以 `NEXT_PUBLIC_MAX_EDIT_SOURCES` 默认设为 16；换到校验更严的端点时把它调小即可。
探测脚本：`scripts/test-multi-image.mjs`，20 秒内给出结论。

多张源图时，**蒙版只作用于第一张**（与 OpenAI 行为一致），画布上会提示这一点。

## 类型检查的坑

`package.json` 里 `typescript` 被 alias 成 `npm:@typescript/typescript6`（TS 6.x），
而 `npm run typecheck` 显式走 `node_modules/typescript-7/bin/tsc`（TS 7.x）。

- **请用 `npm run typecheck`**，不要裸跑 `npx tsc`——两者解析到的版本不同。
- `npm run build`（Next 自带的类型检查）走的是 6.x。两个版本目前都能过，
  但如果将来出现「typecheck 通过、build 报类型错」的怪现象，先怀疑这里。

## 常用命令

```bash
npm run dev              # 开发服务（或双击桌面「GPT Image Playground」图标）
node scripts/test-multi-image.mjs <base-url> <key> [model] [图A] [图B]
                         # 探测某个端点是否真的转发多张参考图（编辑接口）
npm run typecheck        # TS 7 类型检查
npm run lint             # ESLint（当前 0 error / 0 warning）
npm run i18n:check       # 校验每个 t('…') 都有中文词条（改文案后必跑）
npm run check            # lint + typecheck + i18n:check 三合一
npm run build            # 生产构建（已验证可过）

bash scripts/start-playground.sh   # 后台起服务 + 打开浏览器
bash scripts/stop-playground.sh    # 停止服务（等同页面右上角「关闭服务」）
```
