# Kerub — Agent Design

| | |
|---|---|
| **Direction** | *Sage* — sage-tinted neutrals, deep teal accent, Atkinson Hyperlegible |
| **Status** | First design (roadmap P.5); refined in task T3.0 |
| **Files** | [`kerub-agent-ui.png`](kerub-agent-ui.png) (overview, readable on GitHub) · [`kerub-agent-ui.html`](kerub-agent-ui.html) (interactive, open locally in a browser) |

This folder is the reference for the agent's look and behavior. Task T1.12
turns the tokens below into Tailwind theme values; later UI tasks implement
the screens.

---

## 1. Principles

1. **Color is never the only cue.** Every global state has a glyph inside
   the gate shape (✓ ! ×); every severity has a 5-step meter and its name.
2. **The gate shape is the only ornament.** It marks Kerub's own windows and
   carries the global state in the tray icon.
3. **Security phrase first.** Every prompt that asks for the password shows
   the phrase *above* any input, with "Not your phrase? Close this window
   without typing anything."
4. **Audit vs enforce is always visible:** dashed chip for *Audit*, solid
   chip for *Enforce*, everywhere.
5. **Undo sits where Kerub acted** — on the alert row and in the alert
   detail — and asks for the password under the security phrase.
6. **French is ~30 % longer:** no fixed widths on text, label columns sized
   to content, button rows wrap, headings may take two lines.
7. **Accessible:** text contrast ≥ 4.5:1 and meters/glyphs ≥ 3:1 in both
   themes; visible focus ring; every screen usable with the keyboard —
   except the new-keyboard prompt, which is mouse or trusted-keyboard only
   (SEC-DEV-001).

---

## 2. Tokens

Colors are in OKLCH (supported by WebView2 and Tailwind v4).

### 2.1 Neutrals

| Token | Light | Dark |
|-------|-------|------|
| `desk` (behind windows) | `oklch(0.915 0.0108 165)` | `oklch(0.13 0.009 165)` |
| `bg` | `oklch(0.972 0.009 165)` | `oklch(0.185 0.009 165)` |
| `surface` | `oklch(0.995 0.0036 165)` | `oklch(0.215 0.009 165)` |
| `surface-2` | `oklch(0.952 0.0108 165)` | `oklch(0.255 0.009 165)` |
| `border` | `oklch(0.885 0.0144 165)` | `oklch(0.32 0.0108 165)` |
| `border-strong` | `oklch(0.62 0.018 165)` | `oklch(0.56 0.0135 165)` |
| `text` | `oklch(0.23 0.0225 165)` | `oklch(0.955 0.0054 165)` |
| `muted` | `oklch(0.46 0.0225 165)` | `oklch(0.75 0.009 165)` |

### 2.2 Accent (actions, focus)

| Token | Light | Dark |
|-------|-------|------|
| `accent` | `oklch(0.45 0.08 205)` | `oklch(0.80 0.09 200)` |
| `accent-fg` (text on accent) | `#ffffff` | `oklch(0.18 0.03 205)` |
| `accent-soft` | `oklch(0.955 0.02 195)` | `oklch(0.28 0.035 200)` |
| `accent-soft-border` | `oklch(0.86 0.04 200)` | `oklch(0.42 0.055 200)` |
| `accent-on` (accent text on surfaces) | `oklch(0.43 0.08 205)` | `oklch(0.82 0.08 200)` |
| `danger` / `danger-fg` | `oklch(0.53 0.20 27)` / `#ffffff` | `oklch(0.68 0.19 25)` / `oklch(0.17 0.03 25)` |

### 2.3 Global states

Each state has a foreground (`fg`), background (`bg`) and border (`bd`).

| State | Glyph | Light fg / bg / bd | Dark fg / bg / bd |
|-------|-------|--------------------|-------------------|
| Protected | ✓ | `oklch(0.44 0.11 150)` / `oklch(0.955 0.035 150)` / `oklch(0.85 0.07 150)` | `oklch(0.82 0.12 150)` / `oklch(0.28 0.05 150)` / `oklch(0.40 0.08 150)` |
| Degraded | ! | `oklch(0.47 0.10 65)` / `oklch(0.96 0.05 85)` / `oklch(0.86 0.09 80)` | `oklch(0.86 0.12 85)` / `oklch(0.29 0.05 80)` / `oklch(0.42 0.08 80)` |
| Not protected | × | `oklch(0.48 0.18 27)` / `oklch(0.955 0.03 25)` / `oklch(0.85 0.07 25)` | `oklch(0.80 0.12 25)` / `oklch(0.29 0.06 25)` / `oklch(0.42 0.10 25)` |

### 2.4 Severities

Matches `event.severity` 0–4 in [event-schema.md](../event-schema.md) §6.
The meter shows 1 to 5 filled bars.

| Severity | Light fg / bg / bd | Dark fg / bg / bd |
|----------|--------------------|-------------------|
| Info | `oklch(0.45 0.03 255)` / `oklch(0.955 0.008 255)` / `oklch(0.86 0.015 255)` | `oklch(0.84 0.02 255)` / `oklch(0.28 0.01 255)` / `oklch(0.40 0.015 255)` |
| Low | `oklch(0.47 0.11 240)` / `oklch(0.955 0.025 240)` / `oklch(0.86 0.05 240)` | `oklch(0.82 0.09 240)` / `oklch(0.28 0.05 240)` / `oklch(0.40 0.08 240)` |
| Medium | `oklch(0.48 0.10 70)` / `oklch(0.965 0.05 90)` / `oklch(0.87 0.09 85)` | `oklch(0.87 0.12 90)` / `oklch(0.29 0.05 85)` / `oklch(0.42 0.08 85)` |
| High | `oklch(0.50 0.14 48)` / `oklch(0.955 0.04 55)` / `oklch(0.86 0.08 50)` | `oklch(0.82 0.12 50)` / `oklch(0.29 0.06 50)` / `oklch(0.42 0.10 50)` |
| Critical | `oklch(0.47 0.19 25)` / `oklch(0.95 0.035 25)` / `oklch(0.84 0.08 25)` | `oklch(0.80 0.13 25)` / `oklch(0.30 0.07 25)` / `oklch(0.46 0.12 25)` |

### 2.5 Typography

Fonts: **Atkinson Hyperlegible Next** (text) and **Atkinson Hyperlegible
Mono** (identifiers, IDs, technique codes). Both are under the SIL Open
Font License: they are **bundled with the agent** (no remote fonts,
ADR-0002) together with their license file.

| Style | Size / line height | Weight | Example |
|-------|--------------------|--------|---------|
| Title L | 20 / 26 | 600 | Choose your security phrase |
| Title | 18 / 24 | 600 | Approve this USB drive? |
| Emphasis | 16 / 22 | 600 | “Blue heron by the old mill” |
| Body | 14 / 20 | 400 | Its keystrokes are held back until you confirm. |
| Label | 13 / 18 | 500 | Just this time |
| Caption | 12 / 16 | 400 | Blocked automatically in 0:52 if you don't answer. |
| Mono | 12 / 16 | 400 | VID 0781 · PID 5583 · T1110 |

### 2.6 Shape and depth

| Token | Value |
|-------|-------|
| `radius-sm` | 8 px |
| `radius` | 10 px |
| `radius-lg` (windows) | 16 px |
| Shadow, light | `0 1px 2px rgba(20,20,20,.06), 0 12px 32px rgba(20,20,20,.10)` |
| Shadow, dark | `0 12px 32px rgba(0,0,0,.5)` |

---

## 3. Screens

| # | Screen | Key behavior | Requirements |
|---|--------|--------------|--------------|
| 1 | Tray menu (3 states) | State shown by color, glyph and gate outline; admin items marked *Admin* and run through UAC | FR-CORE-002, FR-CORE-010 |
| 2 | USB drive approval | Security phrase first; *Just this time* (until unplugged) or *Always*; password; closing = stays blocked | FR-DEV-001, FR-DEV-002, SEC-AG-001 |
| 3 | New keyboard confirmation | Mouse or trusted keyboard only; composite interfaces flagged; blocks after 60 s | FR-DEV-003, FR-DEV-007, SEC-DEV-001 |
| 4 | First-run setup (4 steps) | Elevated once; password, phrase, profile, audit mode | FR-CORE-008, SEC-IPC-007, SEC-ST-004 |
| 5 | Service unreachable / impersonated | Two tones: degraded vs critical; phrase hidden and password refused when unverified | SEC-SVC-005, SEC-IPC-002, SEC-AG-005 |
| 6 | Dashboard | Global state, module cards with Audit/Enforce chips, recent alerts with Undo | FR-VIS-001, FR-CORE-005 |
| 7 | Alert detail | What happened, why it matters, action taken with Undo, MITRE technique, technical details on demand | FR-VIS-002, FR-VIS-009, FR-RSP-003, FR-RSP-008 |
| 8 | Network card | Profile switch, kill switch states, captive-portal exception (2/5/10 min, password, ends when the VPN connects) | FR-NET-001, FR-NET-004 to FR-NET-008 |

## 4. Not in this design yet

Windows Hello appears in the USB prompt as an alternative confirmation
method; it is planned for later (FR-DEV-008) and will not be implemented
before v1.0. Screens still to design in T3.0: timeline, devices page,
detection page, settings, posture page, maintenance mode.