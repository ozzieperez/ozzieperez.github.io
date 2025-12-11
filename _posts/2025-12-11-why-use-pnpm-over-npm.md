---
layout: post
title: "Monorepo using pnpm"
subtitle:
cover-img: "/assets/img/pnpm-monorepo.png"
thumbnail-img:
comments:
tags:

share-title:
share-description:
share-img:

author:
readtime:
show-avatar: false
social-share: true
nav-short: true
gh-repo: 
gh-badge:
last-updated:


date: 2025-12-11 14:45:19-0500
---

pnpm’s superpower in a monorepo is that it **treats dependencies like a shared, deduped infrastructure layer** instead of a bunch of repeated `node_modules` folders, *and* it gives you powerful workspace tooling on top.

I’ll break it into:

1. **Why pnpm is great for monorepos**
2. **Key concepts / mental model**
3. **Commands you actually want to use day-to-day (with examples)**

---

## 1. Why pnpm in a monorepo?

### a) Massive disk + speed wins

* pnpm keeps a **global, content-addressable store** of packages and then **hard-links** them into each project’s `node_modules`. A given version of `react` is only stored once on disk, no matter how many apps/libs use it.([Refine][1])
* Installs are fast because it mostly just links things that are already in the store and parallelizes resolution/fetch/link steps.([Refine][1])

In a big monorepo you might have 10+ apps and 20+ packages all using the same stack – pnpm shines there.

---

### b) Built-in monorepo support

pnpm has **first-class workspaces**: drop a `pnpm-workspace.yaml` in the repo root and it understands that the repo is a multi-package workspace.([pnpm.io][2])

At the root:

```bash
pnpm install
```

…installs all dependencies for all workspace packages by default (can be turned off with `recursive-install=false` if you ever want).([pnpm.io][3])

No extra “workspace manager” is required for basic linking; it plays nicely with tools like Turborepo/Nx that orchestrate builds on top.

---

### c) Strict dependency isolation (no “phantom deps”)

pnpm’s `node_modules` layout is **non-flat and strict**:

* A package only sees what’s declared in its own `dependencies` / `devDependencies`.
* You can’t accidentally import a transitive dependency that “happens to be hoisted” like with classic npm/Yarn.

This is intentionally enforced to **prevent hidden coupling** and weird runtime bugs; it’s a common “why pnpm for monorepos?” selling point.([JavaScript Development Space][4])

---

### d) Workspace features that scale

A few things that matter a lot once the repo grows:

* **`--filter` selectors**: run/install things only for certain packages (e.g. only apps, only a single service, etc.).([pnpm.io][5])
* **Recursive commands**: `-r` / `--recursive` to run scripts across all packages.
* **Catalogs**: define shared dependency versions in `pnpm-workspace.yaml` so you upgrade one line instead of touching 20 `package.json`s.([pnpm.io][6])
* **Overrides**: force a specific version of a dependency across the entire workspace when you need to hot-fix or unify something.([pnpm.io][7])
* **Docker-optimized flows** with `pnpm fetch` + `pnpm install --offline` – great when building multiple images from a monorepo.([pnpm.io][8])

---

## 2. Core pnpm concepts (mental model)

### 2.1 Global store & virtual store

* **Global store**: content-addressable cache of tarballs, shared across projects.
* **Virtual store**: `.pnpm/` directory inside each project where dependencies live in a deterministic layout, then symlinked into `node_modules`.

You rarely touch this directly, but it explains why:

* Installs stay fast across repos.
* Deleting a project doesn’t delete the underlying cached packages.

([Refine][1])

---

### 2.2 Workspaces: `pnpm-workspace.yaml`

At the root of your monorepo:

```yaml
# pnpm-workspace.yaml
packages:
  - "apps/*"
  - "packages/*"
```

* This file tells pnpm which directories are part of the workspace.([pnpm.io][2])
* From that point:

  * `pnpm install` at root installs everything.
  * Local packages are linked together automatically when referenced via `workspace:` or compatible version ranges.

---

### 2.3 `workspace:` protocol (local linking)

Instead of:

```jsonc
// apps/web/package.json
"dependencies": {
  "@acme/ui": "1.0.0"
}
```

Use:

```jsonc
"dependencies": {
  "@acme/ui": "workspace:*"
}
```

or a specific range:

```jsonc
"dependencies": {
  "@acme/ui": "workspace:^1.0.0"
}
```

That tells pnpm “**use the local workspace package for this**” and keeps versioning/links explicit.

---

### 2.4 Catalogs (centralizing versions)

In a large monorepo, to keep a single source of truth for versions (say `react`, `typescript`, etc.):

```yaml
# pnpm-workspace.yaml
catalogs:
  default:
    react: ^18.3.0
    typescript: ^5.7.0
```

Then in `package.json`:

```jsonc
"dependencies": {
  "react": "catalog:default"
}
```

Now you can bump `react` in one place and all packages following the catalog will pick it up. Helps avoid version drift + merge conflicts when many people touch `package.json`.([pnpm.io][6])

---

### 2.5 Overrides

In `pnpm-workspace.yaml`:

```yaml
overrides:
  "lodash": "^4.17.21"
  "react@^18.0.0": "18.3.1"
```

This **forces** those versions everywhere in the graph (great for security fixes or “we need to align everything on one version”).([pnpm.io][7])

---

### 2.6 Scripts & PATH

* `pnpm run <script>` or simply `pnpm <script>` runs a script from `package.json`.([pnpm.io][5])
* For workspaces, the root’s `node_modules/.bin` is also on `PATH`, so any root-installed tooling (ESLint, Turbo, etc.) is easily accessible from every package.([pnpm.io][5])

---

## 3. Commands & patterns you should know

I’ll keep this to the ones that actually matter in a real monorepo.

### 3.1 Setup & basics

**Install pnpm (modern Node via Corepack):**([freecodecamp.org][9])

```bash
corepack enable
# optional: lock a version globally
corepack use pnpm@latest
```

**Initialize a project:**

```bash
pnpm init
# (no -y flag; pnpm doesn't support `-y` like npm)
```

**Install all deps in the repo:**

```bash
pnpm install               # alias: pnpm i
pnpm install --frozen-lockfile   # CI-style, respects pnpm-lock.yaml strictly:contentReference[oaicite:16]{index=16}
```

**Add a dependency to the current package:**

```bash
pnpm add react
pnpm add -D typescript     # devDependency
```

**Add a dep at the workspace root (shared tooling, etc.):**

```bash
pnpm add -w -D turbo eslint
# -w == --workspace-root
```

---

### 3.2 Workspace-aware dependency management

**Add a dep to a specific package in the monorepo:**

```bash
pnpm add zod --filter ./apps/web
# or by package name (from its package.json "name" field)
pnpm add zod --filter web
```

**Add the same dep to many packages at once:**

```bash
# all apps whose name starts with "web-"
pnpm add @sentry/nextjs --filter "web-*"

# all workspace packages
pnpm add vitest -D -r
```

`--filter` + `-r` are your best friends in big repos.([pnpm.io][5])

---

### 3.3 Running scripts across the monorepo

From the repo root:

```bash
# run "dev" only in the web app
pnpm dev --filter web

# run tests in all packages that have a "test" script
pnpm -r test

# run build in all packages matching a selector
pnpm -r --filter "app-*" build
```

Notes:

* `pnpm run <script>` and `pnpm <script>` are equivalent when `<script>` is defined.
* `pnpm run "/^lint:.*/"` can run multiple scripts that match a regex (e.g. `lint:types`, `lint:styles`).([pnpm.io][5])

---

### 3.4 Exec & tooling

**Run a binary from `node_modules/.bin` via pnpm:**

```bash
pnpm exec tsc --noEmit
# or shorthand:
pnpm tsc --noEmit
```

If the name isn’t a pnpm command and not a script, `pnpm <cmd>` falls back to `pnpm exec <cmd>`.([freecodecamp.org][9])

**Run an arbitrary shell command in every workspace package:**

```bash
pnpm -r exec node -p "process.cwd()"
pnpm -r exec rm -rf node_modules    # for a hard reset in all packages:contentReference[oaicite:20]{index=20}
```

**One-off CLI (like `npx`):**

```bash
pnpm dlx create-next-app my-app
```

([freecodecamp.org][9])

---

### 3.5 Introspection & debugging

**See what’s installed:**

```bash
pnpm list                   # alias: pnpm ls
pnpm list --recursive
pnpm list "eslint-*" --long
```

([pnpm.io][10])

**Why is this dependency here?**

```bash
pnpm why react
pnpm why react --recursive
```

Shows which packages depend on `react`, with a dependency tree.([pnpm.io][11])

**Check for outdated deps:**

```bash
pnpm outdated
```

**Update deps:**

```bash
pnpm up             # respect version ranges
pnpm up --latest    # bump to latest ignoring existing ranges:contentReference[oaicite:24]{index=24}
```

---

### 3.6 Store, cache & pruning

**Show where the store lives:**

```bash
pnpm store path
```

**Clean unused stuff:**

```bash
pnpm prune                   # remove unused packages
pnpm store prune             # prune old data from store
```

([pnpm.io][12])

---

### 3.7 Docker / CI goodies (especially for monorepos)

**Pre-fetch deps using only the lockfile:**

```bash
pnpm fetch --prod
pnpm fetch                   # for all deps
```

Then you can do:

```bash
pnpm install -r --offline --prod
```

in your Dockerfile to make builds very cache-friendly — important when your monorepo feeds several images.([pnpm.io][8])

---

### 3.8 Config that matters for monorepos

In `pnpm-workspace.yaml` and `.npmrc`:

* `recursive-install=false` – if you *don’t* want `pnpm install` at root to install every package.([pnpm.io][3])
* `useNodeVersion: 20.18.0` – pin Node for the whole workspace without `nvm`.([pnpm.io][7])
* `overrides` / `catalogs` – centralize versions and enforce consistency (mentioned above).

---

## If you want to “master pnpm” in your monorepo

If you adopt just a few habits, you’ll get 90% of the value:

1. **Always install from the root** using `pnpm install` (with `--frozen-lockfile` in CI).
2. **Use `pnpm-workspace.yaml` + `workspace:` protocol** for local packages, and keep shared versions in **catalogs**.
3. **Drive everything with `--filter` and `-r`**:

   * `pnpm dev --filter web`
   * `pnpm -r test`
   * `pnpm add zod --filter web`



[1]: https://refine.dev/blog/how-to-use-pnpm/?utm_source=chatgpt.com "A Complete guide to pnpm"
[2]: https://pnpm.io/workspaces?utm_source=chatgpt.com "Workspace"
[3]: https://pnpm.io/cli/install?utm_source=chatgpt.com "pnpm install"
[4]: https://jsdev.space/complete-monorepo-guide/?utm_source=chatgpt.com "Complete Monorepo Guide: pnpm + Workspace + Changesets ..."
[5]: https://pnpm.io/cli/run?utm_source=chatgpt.com "pnpm run"
[6]: https://pnpm.io/9.x/catalogs?utm_source=chatgpt.com "Catalogs"
[7]: https://pnpm.io/settings?utm_source=chatgpt.com "Settings (pnpm-workspace.yaml)"
[8]: https://pnpm.io/cli/fetch?utm_source=chatgpt.com "pnpm fetch"
[9]: https://www.freecodecamp.org/news/how-to-use-pnpm/?utm_source=chatgpt.com "How to Use pnpm – Installation and Common Commands"
[10]: https://pnpm.io/next/cli/list?utm_source=chatgpt.com "pnpm list"
[11]: https://pnpm.io/cli/why?utm_source=chatgpt.com "pnpm why"
[12]: https://pnpm.io/cli/link?utm_source=chatgpt.com "pnpm link"

