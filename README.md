# GitHub Actions Runner Image for Mittwald

This repository builds a self-hosted GitHub Actions runner image for `mw stack` / `mw deploy` based on `myoung34/github-runner:ubuntu-noble`.

The image bakes in:

- Node.js `24.11.1`
- Corepack and `pnpm`
- `ffmpeg`
- Playwright Chromium system dependencies
- Playwright Chromium browser binaries in `/ms-playwright`

## Playwright version pin

The installed browser version follows the `@playwright/test` version pinned in [package.json](/Users/christian/IdeaProjects/nerdhive/github-actions-runner/package.json). Update that dependency when you want the runner image to follow a different Playwright release.

## Build and push

Use the GitHub Actions workflow in [.github/workflows/build-runner-image.yml](/Users/christian/IdeaProjects/nerdhive/github-actions-runner/.github/workflows/build-runner-image.yml).

It:

- can be started manually with `workflow_dispatch`
- also rebuilds on relevant file changes
- logs in to GHCR with `GITHUB_TOKEN`
- pushes
  - `ghcr.io/<owner>/<repo>-github-runner:ubuntu-noble-playwright`
  - `ghcr.io/<owner>/<repo>-github-runner:sha-<commit>`

## Mittwald usage

Use [stack.example.yml](/Users/christian/IdeaProjects/nerdhive/github-actions-runner/stack.example.yml) as a starting point.

Important runtime settings:

- `RUNNER_WORKDIR=/runner/_work`
- `EPHEMERAL=false` if you want a persistent runner instead of one-shot ephemeral registration
- `PLAYWRIGHT_BROWSERS_PATH=/ms-playwright`
- `PNPM_STORE_DIR=/pnpm-store`

The image also includes the upstream runner fix from commit [`7c250c675c186ebaaaa9cd318dfe30f7b3973628`](https://github.com/myoung34/docker-github-actions-runner/commit/7c250c675c186ebaaaa9cd318dfe30f7b3973628), so `EPHEMERAL=false` and `EPHEMERAL=0` correctly disable `--ephemeral`.

Important volume behavior:

- Persist `/runner/_work`
- Persist `/pnpm-store`
- Do not mount `/ms-playwright`, because a volume would hide the browser binaries baked into the image

## CI usage note

Once this image is in use, the workflow running inside the runner should no longer need to execute:

```bash
pnpm --filter @nerdhive/web exec playwright install --with-deps chromium
```

You can keep a lightweight smoke check like this if desired:

```bash
ffmpeg -version
pnpm exec playwright --version
```
