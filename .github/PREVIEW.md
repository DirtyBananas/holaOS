# Preview & Build Pipeline

holaOS is a **packaged Electron desktop app** (Electron 41 + electron-builder 26),
not a hosted web app. There is no GitHub Pages / live URL preview tier, because the
renderer is loaded from the local filesystem inside Electron rather than served over
HTTP. The preview here is a **buildable, downloadable installer artifact**.

Monorepo layout: the root `package.json` delegates to `desktop/` via `npm --prefix desktop`.

## Tiers

### 1. Windows NSIS installer (primary) — `electron-installer.yml`
- Runs on `windows-latest`.
- `npm install --prefix desktop`
- `npm --prefix desktop run dist:win:local`
  - Builds the local sidecar **runtime** from source (`prepare:runtime:local:windows`).
  - Builds the Vite renderer + tsup Electron main.
  - Rebuilds the native dependency `better-sqlite3` for Electron via `@electron/rebuild`.
  - Packages an NSIS `.exe` installer (x64) with electron-builder.
- Uploads the `.exe` as the `holaos-windows-installer` artifact.

### 2. Renderer compile check (fallback) — same workflow, `renderer-fallback` job
- Runs on `ubuntu-latest`.
- `npm --prefix desktop run build:renderer` (output: `desktop/out`).
- Proves the React/Vite renderer compiles even if the native electron-builder
  packaging path fails in a constrained CI/sandbox environment.

Triggers: push to `main` and `claude/repo-organization-artifacts-tnwgtb`, plus manual
`workflow_dispatch`.

## Caveats — please read

- **Unsigned build.** Code signing is intentionally disabled
  (`CSC_IDENTITY_AUTO_DISCOVERY=false`). The installer is not notarized/signed, so
  Windows SmartScreen will warn on launch. Do not distribute this artifact as a
  trusted release.
- **Backend + secrets required to actually run.** The app talks to a remote backend
  (`HOLABOSS_BACKEND_BASE_URL`, e.g. `http://35.160.37.189`) and auth endpoints
  (`api.holaboss.ai`). The installer builds and produces a launchable app, but the
  app **cannot fully function without backend connectivity and credentials**. No
  secrets are configured in CI by design — none are needed to produce the installer.
- **macOS not built here.** Per-platform scripts exist (`dist:mac:local`,
  `dist:mac:dmg:local`) requiring a macOS runner; only the Windows target is wired in
  this pipeline. Linux packaging is not pre-wired as a single script.
- **Native rebuild.** `better-sqlite3` is rebuilt for Electron during `dist:win:local`.
  If that step fails in a constrained environment, rely on the renderer-fallback job
  for a compile-proof artifact.
- **No live web preview / no GitHub Pages.** Not applicable to a desktop Electron app.
