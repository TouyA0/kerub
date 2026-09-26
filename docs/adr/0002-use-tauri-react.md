# ADR-0002: Use Tauri v2 with React for the agent UI

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-09-26 |
| **Deciders** | Quentin (TouyA0) |
| **Related** | TM-AG-2, TM-AG-3, SEC-AG-003, SEC-AG-004, NFR-PERF-002, NFR-USA-002, NFR-USA-004, ADR-0001 |

## Context

- `kerub-agent` needs a tray icon, a dashboard and timeline, approval
  prompts and notifications, in the user session.
- It must stay light (NFR-PERF-002: under 150 MB idle), translatable
  (NFR-USA-002) and accessible (NFR-USA-004), and it should look polished:
  the agent is the only part of Kerub most users will ever see.
- It displays attacker-controlled data (file names, command lines,
  hostnames), so it must resist script injection (TM-AG-3).
- The rest of Kerub is in Rust (ADR-0001); the author is comfortable with
  web front-end development.

## Options considered

### Application framework

- **Web UI served by the service on localhost** — the SYSTEM service would
  expose an HTTP endpoint reachable by every local process and by web pages
  (CSRF, DNS rebinding). Rejected: it widens the attack surface of the most
  sensitive component.
- **Electron** — mature, but ships Chromium and Node.js: high memory usage,
  large attack surface, second runtime.
- **Wails** — good, but its back-end is Go, a second language.
- **Native Rust GUI (egui, iced, Slint)** — pure Rust, but less polished
  results for this kind of dashboard and web skills unused.
- **WinUI through `windows-rs`** — native look, but immature tooling from
  Rust and slow development.
- **Tauri v2** — Rust back-end, front-end rendered by the system WebView2,
  native system tray, small binaries, stable since October 2024. Its
  **capability system** denies by default what the front-end may call.

### Front-end framework

- **Svelte** — small and simple, escapes text by default; smaller ecosystem
  of polished components.
- **React** — largest ecosystem of high-quality components, animation and
  charting libraries; escapes text by default; best supported by AI
  development tools; most widely used in the job market.

## Decision

We will build `kerub-agent` with **Tauri v2** and this front-end stack:

- **React + TypeScript**, bundled with **Vite**;
- **Tailwind CSS** for styling;
- **shadcn/ui** components, built on Radix primitives. Components are copied
  into the repository rather than installed as a package, so their code is
  reviewed and owned like the rest of Kerub.

Rules that come with this decision:

- Tauri and front-end versions are pinned and upgraded deliberately.
- **Minimal capabilities:** the front-end can call only Kerub's own
  commands. No file system, shell, HTTP or other plugin is enabled unless an
  ADR justifies it.
- **The agent holds no authority** (SEC-AG-003): Tauri commands only
  forward requests to the service through `kerub-ipc`.
- **No remote content** is ever loaded (no CDN, no remote fonts); a strict
  Content Security Policy is set in the Tauri configuration.
- **Event data is rendered as text only.** `dangerouslySetInnerHTML` is
  forbidden and enforced by the ESLint rule `react/no-danger`.
- Every UI string comes from translation files (`en.json`, `fr.json`).

## Consequences

**Positive**

- Polished, accessible components and a large ecosystem.
- Web skills reused; fast UI iteration; smooth hand-off from design
  mock-ups.
- Same language as the rest of Kerub on the back-end; much lighter than
  Electron.
- Front-end permissions enforced by the framework, not only by convention.

**Negative**

- WebView2 runtime required: present on Windows 11, usually present on
  Windows 10; the installer must ensure it is installed.
- An npm dependency tree enters the project: lockfile committed, packages
  kept minimal and justified, Dependabot enabled.

**Follow-up**

- ESLint configuration with `react/no-danger` as an error.
- Capability file reviewed in every change that touches the agent.
- Installer configured to install WebView2 when missing.