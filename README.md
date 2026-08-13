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

Renovate (`renovate.json`) tracks this dependency but never auto-merges it — kundenportal's CI depends on this pin matching its own `@playwright/test` version exactly, so a bump here always gets a human look before merge. When you bump it, also bump the matching dependency in the consuming repo (kundenportal) around the same time so the two don't drift apart while waiting on separate Renovate schedules.

## Dependency policy

`renovate.json` mirrors kundenportal's Renovate policy: a few days' delay before non-security PRs open, security/vulnerability updates merge immediately once green, major bumps always need a human, and GitHub Actions/base-image digests stay pinned (`helpers:pinGitHubActionDigests`, `docker:pinDigests`).

**Ubuntu/apt packages (`ffmpeg`, `curl`, `ca-certificates`, `xz-utils`) are not Renovate-trackable.** They're installed unpinned in `Dockerfile.runner`, and Renovate has no datasource for Ubuntu's apt repositories — there's no version string to diff against. Two things stand in for that instead:

- The base image `myoung34/github-runner:ubuntu-noble` is digest-pinned (`docker:pinDigests`), so a Renovate PR appears whenever upstream publishes a new build — a reasonable proxy signal for "the OS layer changed upstream."
- `build-runner-image.yml` also rebuilds on a weekly schedule (`cron: '0 4 * * 1'`) independent of any dependency bump, with `CACHEBUST` busting the apt-install layer's cache so `apt-get update && install` re-resolves against Ubuntu's current noble repo each time. This is what actually keeps `ffmpeg`/`curl`/`ca-certificates` current — not a Renovate PR.

If a specific apt package ever needs a hard version floor (e.g. a CVE fix that must land immediately, not wait for Monday's cron), pin it explicitly in the `apt-get install` line (`ffmpeg=<version>`) and trigger `workflow_dispatch` manually — Renovate still won't track it, but a manual pin makes the requirement visible in the diff.

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

Once the runner image's `@playwright/test` pin matches the consuming repo's exactly, `playwright install --with-deps chromium` in that repo's CI becomes a near-no-op — Playwright checks the version folder already on the persistent `/ms-playwright` volume and skips the download.

Keep the step anyway rather than removing it. The two pins are bumped by two independent Renovate schedules in two different repos, so nothing guarantees they land in the same order at the same time; without this step, a moment of drift between them would run tests against a mismatched Chromium build with no error message. With it, drift just costs one slightly-slower CI step instead of a silent, hard-to-diagnose test failure. A useful smoke check alongside it:

```bash
ffmpeg -version
pnpm exec playwright --version
```
