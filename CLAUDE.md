# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

`AGENTS.md` is the canonical agent rulebook for this repository; the rules below restate and expand it.

## What this project is

DSH Desktop is an open-source Electron desktop client (Windows/macOS) built on DeepSeek Harness (DSH). The core principle: **the desktop itself is a Cordis plugin** (`dsh-plugin-desktop`) composed into the unmodified upstream DSH Host — window, tray, terminal, updates, and work profiles are delivered through the upstream plugin mechanism, never by patching upstream.

Two pinned upstream artifacts, tracked independently in `upstream.json`:
- `deepseek-harness/` — Git submodule pinned to an exact official commit; **read-only from desktop branches**. Upstream pin updates land as a dedicated commit (gitlink + `upstream.json`), always separate from desktop behavior changes.
- `vendor/dsh-runtime/0.1.2-alpha.1/` — vendored published `@deepseek-ai/dsh-*` npm tarballs. Root `package.json` `resolutions` map **every** `@deepseek-ai/dsh-*` package to these `file:` tarballs (some with Yarn patches in `patches/`). Product builds resolve the vendored tarballs, never the submodule source. `yarn check:vendored-runtime` verifies their integrity — never edit tarballs directly.

## Toolchain split (important)

- Outer repo: **Yarn 4.18.0 through Corepack**, Node `^22.19.0 || >=24.0.0`, `nodeLinker: node-modules` (`.yarnrc.yml`).
- `deepseek-harness/` submodule keeps its own **pnpm** workspace. Never run pnpm there directly; use root `upstream:*` scripts (`yarn upstream:version|install|build|build:official|pack:dsh|prepare-runtime`), which enter the submodule via portable shell.
- Workspaces: `dsh-plugin-desktop`, `dsh-community-market`, `dsh-community-fabric`. Fabric is documentation-only (its `check` only verifies docs). Market is a loadable client package that `dsh-plugin-desktop` depends on and that `yarn build` builds first.
- `_deprecated/` is retired code; do not extend it.

## Setup

```sh
git submodule update --init --recursive
corepack yarn install --immutable
```

## Common commands

| Command | Purpose |
| --- | --- |
| `corepack yarn dev` | Build then launch the desktop app (requires a graphical session) |
| `corepack yarn build` | Build market, then desktop |
| `corepack yarn test` | Unit tests (Vitest) for desktop and market workspaces |
| `corepack yarn typecheck` | TypeScript across both product workspaces |
| `corepack yarn check` | Full headless gate: layout/architecture/bilingual/vendored-runtime gates + fabric/market/desktop checks (build, typecheck, tests, Loader smokes, closure, licenses) |
| `corepack yarn workspace dsh-plugin-desktop vitest run tests/<file>.spec.ts` | Run a single test file (add `-t <name>` to filter by test name) |
| `corepack yarn package:dir` / `dist:win` / `dist:win-portable` / `dist:mac` / `dist:mac-smoke` | Packaging (platform-specific; `dist:win` refuses non-Windows/non-x64 hosts and runs its own gate first) |
| `corepack yarn workspace dsh-plugin-desktop verify:notices` | Refresh `THIRD_PARTY_NOTICES.md` — required after production dependency changes, commit the result |

Builds, typechecks, unit tests, and Loader smokes must remain **headless-safe**; graphical launches are explicit only. Keep `yarn check` green before committing.

## Architecture (big picture)

Detailed in `docs/architecture.en.md` and `dsh-plugin-desktop/README.md`.

**Thin Electron host, shared Web carrier.** The Electron main process boots the official DSH Host as a Cordis root; the Host serves the official Web UI over an HTTP/WebSocket carrier (loopback by default; LAN bind only behind an explicit user confirmation). There is **no renderer IPC plugin system and no raw Electron API in the page** — the sandboxed renderer talks only over the same-origin loopback carrier.

**Startup order:** single-instance lock → resolve active profile from Desktop-owned state → launcher provides native runtime, `desktopProfiles` bootstrap, and bundled-pnpm environment → Host Cordis root mounts Loader entries (Desktop services register before third-party entries) → `dsh-base` + `dsh-web-app` + profile bundles compose the carrier → bind → BrowserWindow loads the loopback page → tray created only after the Web surface loads → profile committed as last-known-good.

**Generation model.** Each start creates one `ElectronShellGeneration` that fully owns its `BrowserWindow`, `Tray`, and listeners; every profile or mode switch disposes the current generation before the next starts. Never cache service references, window objects, or subprocess handles across generations; dispose only through the idempotent `release()`. Platform differences (Windows/macOS/Linux) live in `ElectronPlatformStrategy` adapters selected once at startup.

**Profiles and pnpm services.** The active profile's name/dir come only from `desktopProfiles.current` — never inferred from argv, settings, or URLs. `list()` is read-only; `select()` persists a pending target and completes via restart. The launcher-private services (`desktopRuntime`, `desktopPnpmBootstrap`, Electron/Node helpers) are not third-party APIs; the only public contracts are the `dsh-plugin-desktop/profile-service` and `dsh-plugin-desktop/pnpm` subpath exports. `desktopPnpm.run()` is the single package-manager capability (bundled pnpm against the active profile dir, one operation per generation); plugin installs flow through it, and the Market renderer never sends package names or pm commands — the Host resolves normalized identities. Recovery uses three rotating healthy-start checkpoints; there is no install receipt/snapshot/rollback machinery.

**Presentation modes** (`dsh-desktop.mode` in the DSH-home `settings.yaml`, single source of truth): `compatibility` (default; unchanged official UI below an independent 36 px Desktop frame), `extended` (Desktop-owned root layout still hosting official sidebar/conversation/details occupants), `enhanced` (separate root registration, compact internal-caption geometry). Linux supports compatibility only; unsupported modes are rejected, never silently fallen back. Changing mode or material performs an orderly restart — a live generation never hot-swaps Loader rows, slots, or native materials. Compatibility mode must run the upstream default client without overrides; advanced presentation belongs to desktop-owned client plugins that replace documented slots/services through profile composition.

**Native windows are separate windows.** Desktop dialogs (`DesktopDialogWindow`), Recovery, profile creation, and the Setup Wizard are separate sandboxed `BrowserWindow`s built from `dsh-plugin-desktop/src/native-ui/` (React + shadcn, bundled by Vite) — they never render inside the Web Client tree.

**Source layout of `dsh-plugin-desktop/src/`:** flat Host-side modules at the top (`main.ts`/`bin.ts` bootstrap, `electron-runtime.ts`, `electron-shell-generation.ts`, `profile*.ts`, `pnpm.ts`, `updates.ts`, `desktop-terminal.ts`, windows-* confinement modules), `client/` = the Web Client face (mode validation, frames, layout service, settings UI), `native-ui/` = the separate native windows. The package declares both faces: Cordis Host plugin + `dsh.client` metadata (`platform: "web"`, `inject` list, `cordis.patch.yml` bundle patches).

**Packaging.** Electron Builder with `app.asar`; anything that must be physical on disk (pnpm, node-pty, native files) goes under `app.asar.unpacked`. The packaged-runtime gate (`verify-packaged-runtime`) rejects archives missing runtime entries; profile fallback links must never target virtual ASAR paths.

## Repository gates and conventions

- **Layout gate** (`yarn check:layout`) rejects changes to the submodule URL/commit, workspace member list, package-manager boundary, or DSH runtime family. **Architecture gate** (`scripts/market-dependency-direction.mjs`) enforces dependency direction for the market package.
- **Bilingual docs:** documentation ships as `.md` (Chinese) and `.en.md` (English) pairs, tracked by sibling `*.i18n.yaml` files recording git blob hashes of the last confirmed-consistent state. When editing either side, update the other and record both new hashes; `yarn check:bilingual-docs` enforces this.
- **Conventional commits** (`fix(desktop): ...`, `docs: ...`). Commit before major changes of direction; keep submodule pin updates separate from desktop behavior changes.
- **CI** (`.github/workflows/ci.yml`): Node 22.23.2, Corepack, immutable install with recursive submodules. Doc-only PRs pass after the lightweight gates; product changes run the full `yarn check`.
- Design decisions live as bilingual Agent Notes under `.agents/notes/implemented/`. The owning note for repo topology is `.agents/notes/implemented/process/2026-08-15-pinned-upstream-and-isolated-yarn-workspace.md`; read the relevant note before architectural changes.
- On Windows workstations, headless Linux coverage can run in WSL2 with a **Linux** Node/Corepack (not the Windows shims inherited via `/mnt` PATH); see the WSL section of `dsh-plugin-desktop/README.md`.
