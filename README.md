# Blog

Hexo 8 blog using the vendored Icarus theme in `themes/icarus`.

## Requirements

- Node.js 24.18.0 (`.nvmrc`)
- pnpm 11.17.0 (`package.json#packageManager`)

## Development

```sh
pnpm install --frozen-lockfile
pnpm build
pnpm server
```

Hexo output under `public/`, `db.json`, and the deployment checkout under
`.deploy_git/` are generated and must not be committed. This repository does
not configure commit hooks, lint gates, or CI checks.

Create a post with:

```sh
pnpm new "<title>"
```

Pushes to `main` trigger the deployment workflow, which installs dependencies,
builds the site, and deploys it. Do not run `pnpm deploy` unless an explicit
manual deployment is intended.
