# AGENTS.md

本文件适用于整个仓库。

## 项目概览

这是一个 Hexo 6 博客站点，当前主题为本仓库内的 `themes/icarus`。

- 站点源码在 `source/`。
- 文章在 `source/_posts/`。
- 页面/文章脚手架在 `scaffolds/`。
- Hexo 主配置在 `_config.yml`。
- Icarus 主题配置在 `_config.icarus.yml`。
- 本地主题代码在 `themes/icarus/`。
- `public/`、`db.json`、`.deploy_git/` 是生成或部署产物，通常不要手工修改。

## 常用命令

优先使用 pnpm，本项目声明的包管理器是 `pnpm@11.5.0`。

```bash
pnpm install
pnpm build
pnpm server
pnpm clean
pnpm deploy
pnpm new "<title>"
```

说明：

- `pnpm build` 等价于 `hexo generate`，用于验证站点能否生成。
- `pnpm server` 用于本地预览。
- `pnpm clean` 会清理 Hexo 生成缓存和产物。
- 仓库里存在 `yarn.lock`，但除非明确要求迁移或同步锁文件，否则不要更新它。

## 写作约定

- 新文章优先放在 `source/_posts/`。
- 文章文件名沿用现有风格：中文标题可用连字符连接，保留必要英文技术名词。
- front matter 至少包含 `title`、`date`、`tags`。
- 技术文章正文可以使用中文标题结构；已有文章同时存在 `#` 与 `##` 风格，新增内容优先保持同篇文章内一致。
- 引用站内 demo 或图片时，使用 Hexo 渲染后的站内路径，例如 `/demos/...` 或 `/articleImgs/...`。
- 不要把大体积二进制文件直接放进文章目录；如确需示例文件，优先放在 `source/files/` 下的独立子目录。

## 主题与样式

- 主题布局使用 Inferno/JSX，主要在 `themes/icarus/layout/`。
- 主题样式使用 Stylus，主要在 `themes/icarus/source/css/` 和 `themes/icarus/include/style/`。
- 修改主题配置优先改 `_config.icarus.yml`；只有配置无法满足需求时再改 `themes/icarus/` 代码。
- 主题文件来自 Icarus，上游结构尽量保持清晰，避免无关重排或大面积格式化。

## 开发注意事项

- 编辑前先检查当前工作区状态，避免覆盖用户未提交的修改。
- 不要提交或手改 `node_modules/`、`public/`、`db.json`、`.deploy_git/`。
- 对配置、主题布局、样式或依赖有改动时，至少运行 `pnpm build` 验证。
- 对纯文章内容改动，可以不强制构建；如改动 front matter、站内链接、图片路径或 Markdown 语法较多，建议运行 `pnpm build`。
- 保持改动聚焦；不要顺手重写无关文章、主题文件或锁文件。
