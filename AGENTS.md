# 博客 Agent Guide

本文件适用于整个仓库。保持内容精简。

## 项目定位

Hexo 博客站点，主题为本仓库内的 `themes/icarus`。

持久约定：

- 站点源码在 `source/`；文章在 `source/_posts/`。
- 生成/部署产物（`public/`、`db.json`、`.deploy_git/`）不要手工修改或提交。
- 使用 `pnpm@11.17.0`，不要改用 yarn 作为 SoT 锁文件。

## 开始工作前

1. 阅读用户请求与相关源码/配置。
2. 检查 `git status --short`，保留无关改动。
3. 选择最小改动；除非用户明确要求，不要 commit/push/deploy。

## 仓库地图

- `source/`：站点内容
- `source/_posts/`：文章
- `scaffolds/`：脚手架
- `_config.yml` / `_config.icarus.yml`：站点与主题配置
- `themes/icarus/`：本地主题
- `public/`、`db.json`、`.deploy_git/`：生成或部署产物

## 编码约定

- 语言包：typescript-node（轻量；主要为 Hexo/Node 工具链）。
- 用户可见文章以中文为主；配置键与命令保持英文。

## 运行时与环境

- Node.js `>=24.18.0 <25`、`.nvmrc` `24.18.0`、`pnpm@11.17.0`（兼容范围 `>=11.17.0 <12`）。
- `themes/icarus/package.json#engines` 是 vendored 主题的消费者兼容元数据，不是本仓库的开发工具链版本。

## 命令与验证

```bash
pnpm install
pnpm build      # Hexo build；配置/主题/依赖改动后的 handoff
pnpm server
pnpm clean
pnpm deploy
pnpm new "<title>"
```

| 变更 | 必要检查 |
| --- | --- |
| 配置 / 主题 / 依赖 | `pnpm build` |
| 纯文章内容 | `git diff --check`；链接/front matter 改动较多时建议 `pnpm build` |
| 任意已跟踪文件 | 不提交 `node_modules/`、`public/`、`db.json`、`.deploy_git/` |

## 写作约定

- 新文章优先 `source/_posts/`；front matter 至少含 `title`、`date`、`tags`。
- 站内资源用渲染后路径（如 `/demos/...`）；大体积二进制优先 `source/files/` 子目录。

## 主题与样式

- 布局 Inferno/JSX：`themes/icarus/layout/`；样式 Stylus：`themes/icarus/source/css/` 等。
- 优先改 `_config.icarus.yml`；避免对上游主题大面积无关格式化。

## Git 与提交

- Conventional Commits：`feat` `fix` `docs` `refactor` `perf` `test` `build` `ci` `chore` `revert`。
- 标题 ≤100 字符；不要盲目 `git add -A`。
- 本仓不配置 Git hooks、lint 或 CI 门禁；推送到 `main` 后仅运行部署工作流。

## 完成定义

- [ ] 行为完成且不破坏生成/部署约定
- [ ] 必要检查已跑或披露跳过
- [ ] 交接列出文件、验证与风险
