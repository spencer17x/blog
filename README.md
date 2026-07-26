# Blog

Hexo 8 blog using the vendored Icarus theme in `themes/icarus`.

## Requirements

- Node.js 24.18.0 (`.nvmrc`)
- pnpm 11.17.0 (`package.json#packageManager`)

## Development

```sh
pnpm install --frozen-lockfile
pnpm check
pnpm server
```

`pnpm check` lints the theme, cleans generated state, and builds the site. Hexo
output under `public/`, `db.json`, and the deployment checkout under
`.deploy_git/` are generated and must not be committed.

This content repository intentionally has no TypeScript sources, unit-test
framework, or repository-wide formatter: a global formatter would rewrite
historical posts and the vendored theme. Theme ESLint plus a clean full-site
build is therefore the authoritative local and CI check.

Create a post with:

```sh
pnpm new "<title>"
```

Deployment is performed by GitHub Actions after the same check succeeds on
`main`. Do not run `pnpm deploy` unless an explicit manual deployment is
intended.
