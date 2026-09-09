# UI/UX Technical Design Specification

**Product:** Comms Network Monitor — Autonomous Fleet Control Room
**Document type:** Reverse-engineered technical design specification (source of truth for UI/UX)
**Audience:** Claude Code and any developer adding features to this codebase
**Version:** 2.0 — regenerated from the current code and running preview
**Status of values:** every number in this document was read from source or measured in the running app. Anything that could not be established is marked `UNKNOWN / NEEDS VERIFICATION`.

> This document describes the system **as it is**. It is not a redesign. No UI was changed while producing it.

---

## Table of Contents

1. [How to use this document](#1-how-to-use-this-document)
2. [System overview and technology](#2-system-overview-and-technology)
3. [Screen inventory and routes](#3-screen-inventory-and-routes)
4. [Screen 1 — Tactical view](#4-screen-1--tactical-view)
5. [Screen 2 — Logical view](#5-screen-2--logical-view)
6. [Persistent region — Fleet sidebar (left)](#6-persistent-region--fleet-sidebar-left)
7. [Persistent region — Control Room panel (right)](#7-persistent-region--control-room-panel-right)
8. [Overlays — dialogs, popovers, floating windows](#8-overlays--dialogs-popovers-floating-windows)
9. [Screenshots and annotations](#9-screenshots-and-annotations)
10. [Design system — tokens](#10-design-system--tokens)
11. [Component specifications](#11-component-specifications)
12. [Form field specifications](#12-form-field-specifications)
13. [Tabular / matrix data specifications](#13-tabular--matrix-data-specifications)
14. [States catalogue](#14-states-catalogue)
15. [UX flows](#15-ux-flows)
16. [Loading, empty and error states](#16-loading-empty-and-error-states)
17. [Responsive behaviour](#17-responsive-behaviour)
18. [Accessibility](#18-accessibility)
19. [Code mapping](#19-code-mapping)
20. [Design inconsistencies / existing UX debt](#20-design-inconsistencies--existing-ux-debt)
21. [Rules for new features](#21-rules-for-new-features)
22. [Feature implementation template](#22-feature-implementation-template)
23. [Developer handoff — Claude Code](#23-developer-handoff--claude-code)

---

## 1. How to use this document

- Sections 10–14 are the **binding** part: tokens, component contracts, states. Do not deviate.
- Sections 4–8 describe existing screens; copy their patterns for new screens.
- Section 19 maps every visual element to a file so you can find the real component instead of writing a new one.
- Section 20 lists known inconsistencies and the pattern that should win when you touch that area.

Companion file: `UISPEC.md` (condensed, prescriptive build rules). Where the two disagree, this document is newer.

---

## 2. System overview and technology

| Item | Value |
| --- | --- |
| Framework | TanStack Start v1 (React 19), Vite 7 |
| Router | TanStack Router, file-based (`src/routes/`) |
| UI library | MUI v7 (`@mui/material`, `@mui/icons-material`) |
| Styling | MUI `styled()` + theme only. No Tailwind classes in components. |
| Charts | Recharts (`LineChart`) |
| Theme | Single dark theme, `src/theme/tacticalTheme.ts` |
| Data | Static in-memory modules under `src/data/`. No backend, no API calls, no auth. |
| Language of UI | English, uppercase-heavy tactical wording |
| Text direction | LTR only |

**Domain vocabulary (use these words, no synonyms):**

| Term | Meaning |
| --- | --- |
| CONTROL ROOM | The command post / HQ node |
| HALO | The modem/router hardware family. The words "MODEM" and "ROUTER" are not used in UI copy. |
| HALO CONVOY 23 | The Control Room's HALO unit |
| HALO SERVER | The central aggregation node in the logical view |
| PLATFORM n | A vehicle / asset |
| RELAY | A standalone relay unit |
| Range | A communication medium: `CELLULAR`, `SATCOM`, `RADIO` |
| Mesh Links | The tactical map's link overlay |
| Quality | 0–100 link score |
| Fault badge | Red hollow count badge on a platform |

---

## 3. Screen inventory and routes

There is exactly **one route**.

| # | Screen | Route | Purpose | Users | Primary actions |
| --- | --- | --- | --- | --- | --- |
| 1 | Monitor console — Tactical mode | `/` (`mode = "tactical"`, default) | Geographic situational picture of fleet connectivity | Comms operator in the control room | Select platform/relay, pan/zoom map, toggle Mesh Links, drag RELAY and CONTROL ROOM markers, open camera feed, read alerts |
| 2 | Monitor console — Logical mode | `/` (`mode = "logical"`) | Topology picture: who reaches the Control Room and through whom | Same operator | Select platform, open Control Room panel, read per-link download rates |

Mode is **local component state** in `src/routes/index.tsx` (`useState<ViewMode>`), not a URL segment. Deep links to the logical view are not possible today (see §20).

Route metadata (`src/routes/index.tsx`):

```
title:            "Comms Network Monitor — Autonomous Fleet Control Room"
description:      "Operator console for monitoring link quality, modems, SATCOM and radio ..."
og:title          "Comms Network Monitor — Control Room"
og:type           website
twitter:card      summary_large_image
```

### 3.1 Global UI inventory

| Pattern | Present? | Where |
| --- | --- | --- |
| Modals / dialogs | Yes | Settings dialog, Confirm power change, Action blocked, Coordinate dialog |
| Drawers | No side drawer component in use | `ControlRoomDrawer` exists in the tree but is not mounted by `index.tsx` (dead code — see §20) |
| Popovers | Yes | Notification tray, camera quality menu |
| Floating windows | Yes | Camera window (modal-like, portal), Halo compare windows (draggable) |
| Forms | Yes | Settings dialog only |
| Data tables | Grid-like lists, not `<table>` | Fleet sidebar lists, Link Matrix |
| Filters | No | — |
| Search | No | — |
| Pagination | No | — |
| Tabs | Yes | Settings dialog (4 tabs) |
| Dropdowns / selects | Yes | Settings selects, camera quality menu |
| Toasts / snackbars | No | Alerts are delivered through the notification tray only |
| Empty states | Not implemented (data is always present) | — |
| Loading states | Only the REBOOT countdown | — |
| Error states | No error surface | — |
| Confirmation dialogs | Yes | Every power toggle |

---

## 4. Screen 1 — Tactical view

**Screenshot:** `screenshots-v2/01-tactical-default.png`

### 4.1 Screen information

| Field | Value |
| --- | --- |
| Name | Tactical view (map mode) |
| Route | `/` with `mode="tactical"` |
| Component | `src/components/TacticalMap/TacticalMap.tsx` |
| Purpose | Show where each platform is and how well it is connected |
| UX summary | A satellite raster fills the viewport. Vehicles are icon + compact card markers. Coloured dashed lines represent radio/mesh links. All chrome floats over the map. |

### 4.2 Layout (top-to-bottom, left-to-right)

Three fixed columns (`MonitorLayout` = `display:flex; height:100vh; overflow:hidden`):

```
┌─ Fleet sidebar 296px ─┬────────── Viewport (flex:1) ──────────┬─ Control Room 340px ─┐
│ header 
│ VEHICLES list         │  coords chip (top-left, 12px inset)   │  CONTROL ROOM header │
│ RELAYS list           │  Mesh Links button + bell (top-right) │  HALO panel          │
│ LINK MATRIX           │  map canvas (pan/zoom)                │  RADIO panel         │
│                       │  legend (bottom-left, conditional)    │  monitoring chart    │
│                       │  scale bar + zoom stack (bottom-right)│  compare list        │
│                       │                                       │  footer controls     │
└───────────────────────┴───────────────────────────────────────┴──────────────────────┘
```

| Region | Spec |
| --- | --- |
| Left column | `296px` fixed, `panel.surface #121212`, `border-right 1px divider`, own vertical scroll |
| Center | `flex: 1; min-width: 0; height: 100%; position: relative` |
| Right column | `340px` fixed, `panel.surface`, `border-left 1px divider`, own vertical scroll, sticky footer |
| Top-right bar | `position:absolute; right:12px; top:12px; gap:6px; z-index:5` |
| Coordinate chip | `position:absolute` top-left, opens the coordinate dialog |
| Zoom controls | bottom-right vertical stack: `+`, `−`, recenter |
| Scale bar | bottom-right, label default `500 m` |
| Legend | bottom-left, **rendered only while Mesh Links is ON** |

### 4.3 Map elements

| Element | Behaviour |
| --- | --- |
| Map canvas | CSS `transform: translate(x,y) scale(zoom)`; drag to pan (`useMapViewport`), buttons to zoom, recenter resets |
| Grid overlay | SVG pattern, 6.25% cells, `divider` stroke `0.08`, opacity `0.5` |
| Platform marker | Truck icon badge (blue frame, matching the settings blue) + compact `PlatformCard` below it |
| Relay marker | Hub icon badge, draggable (`useMapDrag`) |
| Control Room marker | Antenna icon badge, draggable, label `CONTROL ROOM` |
| Off-screen indicator | When the Control Room marker is panned outside the viewport, an arrow is projected onto the viewport border pointing at it |
| Marker de-overlap | Markers within `Δx<7%` and `Δy<9%` are fanned horizontally by `52px` steps and `16px` vertical steps |
| Link lines | Dashed SVG paths, colour = `status.good/marginal/poor` |
| Link visibility | `linksOn` shows all; with links OFF, hovering a marker reveals only that marker's links |
| Fault badge | Hollow red ring, floats `top:-8px; right:-6px` relative to the card |

### 4.4 Actions

| Action | Trigger | Result |
| --- | --- | --- |
| Select platform | Click sidebar row or map card | Card gets a 2px primary ring, map recentres on it, right panel switches to that platform |
| Deselect | Click the same row again | Right panel returns to CONTROL ROOM |
| Focus relay | Click a RELAYS row | Map recentres on the relay marker |
| Toggle Mesh Links | `MESH LINKS` button (`aria-pressed`) | Shows/hides all link lines **and** the legend |
| Open camera | Camera icon button on a platform card | Camera window (portal) |
| Open coordinates | Coordinate chip | `CoordinateDialog` |
| Read alerts | Bell button (badge = alert count) | Notification popover |

---

## 5. Screen 2 — Logical view

**Screenshot:** `screenshots-v2/11-logical-default.png`

### 5.1 Screen information

| Field | Value |
| --- | --- |
| Name | Logical view (topology) |
| Route | `/` with `mode="logical"` |
| Component | `src/components/LogicalTopology/LogicalTopology.tsx` |
| Purpose | Show the routing hierarchy and each hop's quality and download rate |

### 5.2 Layout

Vertical three-row hierarchy centred in the viewport (`LayerStack`: `width:100%`, `max-width:1680px`, `min-width:720px`, `height:100%`, `min-height:460px`, `gap: 12px`, `padding: 8px 8px 52px`):

```
row 1   CONTROL ROOM  (80% width, min 600px, transparent fill, 2px primary border)
            └── HALO CONVOY 23 · n PORTS   (rectangle, min-width 260px, detached, margin-bottom 8px)
            │  individual vertical SVG trunks, one per platform route
row 2   HALO SERVER  (50% width, min 420px, 1px accent border, accent tint 10%)
            │  one SVG edge per platform, plus peer-relay edges between platforms
row 3   PLATFORMS  — single non-wrapping row, justify: space-evenly, column-gap 6px
        [PLATFORM 1] [PLATFORM 2] [PLATFORM 3] [PLATFORM 4] [PLATFORM 5]
legend  bottom-left: DIRECT LINK / VIA PEER PLATFORM / GOOD / FAIR / POOR
```

### 5.3 Routing rules (implemented logic — do not change without asking)

1. **Every** platform reaches the Control Room through `HALO SERVER`. No platform draws a line straight to the Control Room.
2. A platform without direct visibility is routed **platform → peer platform → HALO SERVER**; no direct edge is drawn for it.
3. Vehicle→node and node→Control Room segments carry **independent** status colours.
4. Trunks from HALO SERVER to the Control Room are **not merged**: one line per served route, terminating at the HALO CONVOY 23 rectangle. The Control Room box is intentionally `background: transparent` so those lines remain visible inside it.
5. Every segment is labelled with its download rate, format `↓ 46.5 Mbps`.
6. Solid line = direct link. Dashed line = via peer platform.

### 5.4 Node cards

| Node | Size | Content |
| --- | --- | --- |
| Control Room | 80% width / min 600px, padding `10px 16px`, font `1rem`, letter-spacing `0.18em` | Icon + `CONTROL ROOM`, contains the HALO rectangle |
| HALO rectangle | min-width 260px, padding `4px 8px`, `0.8125rem` | `HALO CONVOY 23 · n PORTS`. Sized deliberately small so more HALO units can be added later. |
| HALO SERVER | 50% width / min 420px, padding `10px 16px`, `0.875rem` | `HALO SERVER · n CONNECTED PLATFORMS · n VIA PEER RELAY` |
| Platform card | `PlatformCard variant="topology"`, width `176px`, font `0.75rem` | Full name, full range names, quality bar with `%`, camera button, fault badge |

### 5.5 Interaction

| Action | Result |
| --- | --- |
| Click a platform card | Selects it; its route is highlighted, all other routes dim to `opacity 0.3`; right panel shows the platform |
| Click the Control Room node | Right panel returns to CONTROL ROOM. The rest of the app is **not** dimmed and no backdrop appears. |
| Click empty canvas | Clears selection and resets the view to the Control Room |
| Hover a platform | Tooltip with the full unit name (600ms enter delay) |

The `MESH LINKS MAP` mini-wheel (`ConnectivityWheel`) renders only in this mode, positioned right-of-centre so it does not collide with the right panel. The `MESH LINKS` toggle button is hidden in this mode.

---

## 6. Persistent region — Fleet sidebar (left)

**Component:** `src/components/FleetSidebar/FleetSidebar.tsx` · width `296px`

| Block | Spec |
| --- | --- |
| Header | `panel.header #212121`, padding `12px`, `border-bottom: 2px solid primary@50%`. Title `COMMS NETWORK MONITOR`, `0.875rem / 700 / 0.2em`, colour `primary.main`, rendered as the page `<h1>`. Collapse button on the right (`aria-label="Collapse panel"`). |
| Section headers | `SectionHeader` — centred, `panel.header`, padding `7px 12px`, top+bottom `1px divider`, `0.75rem / 800 / 0.2em`, uppercase |
| VEHICLES list | Row grid: name `96px` · range `74px` · quality `flex:1` |
| RELAYS list | Same column contract |
| LINK MATRIX | `MeshMatrix` — metric checkboxes + 7×7 value grid + frequency footer + colour legend |
| Collapsed rail | `32px` wide, vertical `writing-mode: vertical-rl` title in `primary.main`, whole rail is a button (`aria-label="Expand panel"`) |

Row states: default transparent; hover `primary@8%`; selected `primary@12%` + `2px` left border `primary.main`; selected+hover `primary@18%`.

---

## 7. Persistent region — Control Room panel (right)

**Component:** `src/components/ControlRoomPanel/ControlRoomPanel.tsx` · width `340px`

Order of blocks:

1. **Title** — `SectionHeader emphasis`: `CONTROL ROOM`, or the selected platform's name. `0.875rem / 800 / 0.24em`, colour `primary.light`.
2. **HALO panel** (`ModemPanel`) — collapsible.
3. **RADIO panel** (`RadioPanel`) — power toggle, no expander (the radio has no sub-menu; the chevron was deliberately removed).
4. **Monitoring header** — `CONTROL ROOM MONITORING` or `HALO MONITORING`.
5. **MonitoringGraph** — Recharts line chart.
6. **COMPARE HALO MONITORING** (Control Room context only) — one checkbox row per comparable unit; checking one opens a draggable compare window.
7. **Footer** (sticky, `panel.header`, `border-top 1px divider`, padding `10px 12px`, gap `6px`):
   - Row 1 (full width): `ViewModeSwitch` — `TACTICAL` / `LOGICAL`.
   - Row 2: `CONTROL ROOM` button (active state = primary tint 14% + primary border) · spacer · `SETTINGS` button.

### 7.1 HALO panel (`ModemPanel`) — detail

Collapsed content:

| Row | Content |
| --- | --- |
| Header | Chevron collapse button (`aria-expanded`), cell icon, unit name (e.g. `HALO CONVOY 23`) |
| Meter row | `QualityMeter` (bar with `%` inside) + `14.2 Mbps (RX)` tinted by `rateStatus` |
| Health row | `HealthMetrics dense`: `52°C`, `41%` — outline chips, colour by threshold |

Expanded content adds one row per channel:

`[power toggle] [channel name (+chevron if RF metrics)] [state chip] [signal bars] [rate]`

- State chips are abbreviated so each row stays on one line: `CONN`, `NO CONN`, `NO SIM`, `UNPLG`; the full wording is the chip's `title`.
- Channels that are not `connected`/`disconnected` cannot be powered; their bars render muted grey, never red.
- Expanding a channel with RF data reveals `SINR dB`, `RSSI dBm`, `RSRP dBW/m²` cells.
- A `REBOOT` button appears at the bottom while expanded. During the 10s cycle all panel content dims (`DimWrap`), toggles are disabled, and the button shows `REBOOTING ns` with a sweeping progress fill.

---

## 8. Overlays — dialogs, popovers, floating windows

### 8.1 Settings dialog

**Screenshots:** `07-settings-system.png`, `08-settings-radio.png`

| Property | Value |
| --- | --- |
| Trigger | `SETTINGS` button in the right footer |
| Width | `min(760px, 92vw)`; height auto |
| Position | MUI centred modal, standard MUI backdrop |
| Radius / border | `4px`, `1px solid primary@45%`, glow `0 0 40px primary@20%` |
| Header | `panel.header`, icon + `SYSTEM SETTINGS` (`0.875rem/700/0.24em`, primary) + close icon button |
| Tabs | `System`, `Satellite`, `Radio`, `Cellular` — `0.75rem/600/0.18em`, indicator `primary.main` |
| Footer | Right-aligned `Cancel` (text) + `Apply` (contained) |
| Close | Close icon, `Cancel`, ESC and backdrop click (MUI defaults) |
| Validation | None beyond native `type="number"` + `min/max/step` on the frequency field |
| Success behaviour | `Apply` parses the frequency, pushes it up, closes. No toast, no persistence. |

### 8.2 Confirm power change

**Screenshot:** `04-confirm-power-dialog.png`

| Property | Value |
| --- | --- |
| Trigger | Any `PowerToggle` click (`confirm` defaults to true) |
| Size | `maxWidth="xs"`, `fullWidth` |
| Title | `CONFIRM POWER CHANGE` (`0.8rem`, `0.12em`) |
| Body | `Are you sure you want to turn this communication component OFF/ON? (label)` |
| Actions | `Cancel` (inherit) · `Confirm` (contained) |
| Blocked variant | Title `ACTION BLOCKED`; body is a warning `Alert` (outlined): last remaining active range cannot be switched off; only a `Close` button is shown |

### 8.3 Notification tray

**Screenshot:** `06-notifications-open.png` — bell with red count badge (`Badge color="error"`), opens a `Popover` list of `title / detail / time`, severity-coloured. Static data from `src/data/notifications.ts`. No read/dismiss actions.

### 8.4 Camera window

**Screenshot:** `09-camera-window.png` — portal to `document.body`, dark backdrop, framed window: header `PLATFORM n · ROAD SIM` + quality button (`H/M/L` menu) + close; static feed image; footer `LIVE SIMULATION` / `QUALITY x`. Closes on backdrop click or close button. **ESC does not close it** (see §20).

### 8.5 Halo compare windows

Draggable floating charts opened from `COMPARE HALO MONITORING`. Initial position `x = 120 + 32·index`, `y = 120 + 32·index`. Closing unchecks the source checkbox.

### 8.6 Coordinate dialog

Opened from the map coordinate chip; lets the operator set marker coordinates. Fields: `UNKNOWN / NEEDS VERIFICATION` beyond `CoordinateDialog.tsx` (13-line style file, small MUI dialog).

---

## 9. Screenshots and annotations

All images live in `screenshots-v2/`. Captured at `1600×1000` unless noted.

### Screenshot 1 — Tactical view, default (`01-tactical-default.png`)
1. Sidebar header + collapse button
2. `VEHICLES` list with quality bars
3. `RELAYS` list
4. `LINK MATRIX` (metric checkboxes, 7×7 grid, frequency, legend)
5. Coordinate chip
6. `MESH LINKS` toggle (active) and notification bell with count
7. Map markers: icon badge + compact card
8. Link lines colour-coded by status
9. Link legend (bottom-left, visible because Mesh Links is ON)
10. Zoom controls and scale bar
11. Right panel: title, HALO panel, RADIO panel, chart, compare list, footer

### Screenshot 2 — Mesh Links OFF (`02-tactical-mesh-off.png`)
Link lines and the bottom-left legend are both gone; the button loses its filled primary state. Hovering a marker still reveals that marker's own links.

### Screenshot 3 — HALO panel expanded (`03-halo-panel-expanded.png`)
1. Chevron in expanded state
2. Channel rows `SIM 1…SIM 4`, `ONEWEB`, `STARLINK`
3. State chips `CONN` / `NO CONN` / `NO SIM` / `UNPLG`
4. Signal bars, muted for inactive channels
5. Per-channel rate `12.4 (RX)` or `—`
6. `REBOOT` button

### Screenshot 4 — Confirm power change (`04-confirm-power-dialog.png`)
Compact centred dialog, `Cancel` / `Confirm`.

### Screenshot 5 — Platform selected (`05-platform-selected.png`)
Sidebar row highlighted, map recentred, right panel retitled to the platform, chart gains the latency series and right-hand `ms` axis, compare list hidden.

### Screenshot 6 — Notifications open (`06-notifications-open.png`)
Popover anchored under the bell; four static alerts.

### Screenshot 7 — Settings, System tab (`07-settings-system.png`)
Header, tab bar, `Precheck` group with `Run` button and hint, footer actions.

### Screenshot 8 — Settings, Radio tab (`08-settings-radio.png`)
Frequency (number), channel bandwidth (select), mesh network ID (text), TX power (number).

### Screenshot 9 — Camera window (`09-camera-window.png`)
Framed feed with header, quality control and footer status line.

### Screenshot 10 — Sidebar collapsed (`10-sidebar-collapsed.png`)
Left column reduced to a `32px` rail with vertical title; viewport grows.

### Screenshot 11 — Logical view (`11-logical-default.png`)
1. Control Room box with the HALO CONVOY 23 rectangle inside
2. Individual trunks reaching the rectangle
3. `HALO SERVER` node with its counters
4. Platform row (single line, full names)
5. Per-segment `↓ Mbps` labels
6. `MESH LINKS MAP` wheel
7. Legend, bottom-left

### Screenshot 12 — Logical, platform selected (`12-logical-platform-selected.png`)
Selected route stays at full opacity; all other routes and nodes dim.

### Screenshot 13 — Tablet 1024px (`13-tablet-1024.png`)
Three columns still fixed; the map viewport is squeezed to roughly `1024 − 296 − 340 = 388px`.

### Screenshot 14 — Mobile 390px (`14-mobile-390.png`)
The three-column shell does not reflow: the sidebar consumes the screen and the other columns are cut off. Documented as UX debt, not as intended behaviour.

---

## 10. Design system — tokens

Everything below is defined in `src/theme/tacticalTheme.ts`. **Never hardcode a hex value in a component.**

### 10.1 Colours

| Token | Value | Usage |
| --- | --- | --- |
| `palette.primary.main` | `#90CAF9` | Titles, active controls, links |
| `palette.primary.light` | `#BBDEFB` | Emphasised section title |
| `palette.primary.dark` | `#2477E8` | Hover of filled primary |
| `palette.primary.contrastText` | `#121212` | Text on filled primary |
| `palette.secondary.main` | `#C28FF4` | Latency series |
| `palette.error.main` | `#F44336` | Errors, fault badges |
| `palette.warning.main` | `#FFA726` | Warnings |
| `palette.info.main` | `#29B6F6` | Info |
| `palette.success.main` | `#66BB6A` | Success |
| `palette.background.default` | `#303030` | App background, empty bar track |
| `palette.background.paper` | `#424242` | Cards, dialogs |
| `palette.panel.surface` | `#121212` | Side panels |
| `palette.panel.header` | `#212121` | Panel/section headers, footers |
| `palette.panel.drawer` | `#292929` | Drawer surface |
| `palette.interactive.main` | `#2477E8` | Active/connected accents |
| `palette.interactive.glow` | `rgba(47,128,237,0.4)` | Glow around active accents |
| `palette.status.good` | `#66BB6A` | Quality ≥ 70 |
| `palette.status.marginal` | `#FFA726` | Quality 40–69 |
| `palette.status.poor` | `#F44336` | Quality < 40 |
| `palette.text.primary` | `#FFFFFF` | Body text |
| `palette.text.secondary` | `rgba(255,255,255,0.7)` | Labels, muted values |
| `palette.text.disabled` | `rgba(255,255,255,0.5)` | Disabled text |
| `palette.divider` | `rgba(255,255,255,0.12)` | All borders |
| `action.hover` / `action.selected` | `rgba(255,255,255,0.08)` | Hover / selected fills |
| `action.disabled` | `rgba(255,255,255,0.3)` | Disabled foreground |
| `action.disabledBackground` | `rgba(255,255,255,0.12)` | Off toggle track |

Tints are produced with MUI `alpha(color, n)` — the recurring values are `0.08`, `0.12`, `0.14`, `0.18`, `0.22`, `0.3`, `0.6`, `0.9`.

### 10.2 Status thresholds (`src/lib/linkStatus.ts`)

| Function | good | marginal | poor |
| --- | --- | --- | --- |
| `qualityStatus(q)` | ≥ 70 | 40–69 | < 40 |
| `rateStatus(mbps)` | < 18.7 | ≥ 18.7 (85% of cap) | > 22 (`MAX_BANDWIDTH_MBPS`) |
| `temperatureStatus(°C)` | < 55 | 55–64 | ≥ 65 |
| `cpuStatus(%)` | < 70 | 70–84 | ≥ 85 |
| `voltageStatus(V)` | 11.9–13.5 | 11.5–11.9 or 13.5–14 | < 11.5 or > 14 |

Mesh margin (`ConnectivityWheel`): ≥16 good, ≥10 marginal, else poor.

### 10.3 Typography

Font stack: `"Exo", "Assistant", Arial, sans-serif`. Base `fontSize: 12`. Global `letter-spacing: 0.01em` on `body`.

| Role | Size | Weight | Letter-spacing | Where |
| --- | --- | --- | --- | --- |
| Page title (h1) | `0.875rem` | 700 | `0.2em` | Sidebar header |
| Panel title (emphasis) | `0.875rem` | 800 | `0.24em` | Right panel title |
| Section header | `0.75rem` | 800 | `0.2em` | All `SectionHeader`s |
| Dialog title | `0.875rem` | 700 | `0.24em` | Settings |
| Body1 | `0.875rem` | 400 | — | Default text |
| Body2 / caption | `0.75rem` | 400 | — | Labels, hints |
| Button | `0.75rem` | 600–700 | `0.12em` | Uppercase by theme |
| Metric chip | `0.75rem` (dense) / `0.8125rem` | 500 good / 700 alert | `0.04em` | Health chips |
| Quality bar value | `0.8125rem` | 700 | `0.04em` | Inside the bar |
| Card title (map) | `0.625rem × 1.05em` | 800 | `0.06em` | Overlay cards |
| Card title (topology) | `0.75rem × 1.05em` | 800 | `0.06em` | Logical cards |
| Smallest allowed | `0.625rem` | — | — | Badges only |

All numeric readouts use `font-variant-numeric: tabular-nums`.

### 10.4 Spacing

`theme.spacing = 4px`. `theme.spacing(n) = 4n`.

| Step | px | Typical use |
| --- | --- | --- |
| 0.25–0.75 | 1–3 | Inside badges/chips |
| 1 | 4 | Icon gaps, tight rows |
| 1.5 | 6 | Control gaps |
| 2 | 8 | Row gaps, list padding |
| 2.5 | 10 | Panel block padding |
| 3 | 12 | Panel/header padding, section gaps |
| 4 | 16 | Node padding in the topology |
| 10–13 | 40–52 | Reserved bottom clearance in the topology |

### 10.5 Radius

| Element | Radius |
| --- | --- |
| Buttons, icon buttons, inputs, chips, badges, cards, dialogs (`shape.borderRadius`) | `4px` |
| `MuiPaper` rounded / MUI dialog paper default | `8px` |
| Quality bar track and fill, power toggle track, alert badge | `999px` (pill) |
| Power toggle thumb | `50%` |

### 10.6 Shadows and glows

| Use | Value |
| --- | --- |
| Dialog | `0 8px 18px rgba(0,0,0,0.35)` |
| Settings dialog glow | `0 0 40px primary@20%` |
| Control Room node glow | `0 0 18px primary@35%` |
| HALO SERVER glow | `0 0 14px accent@18%` |
| Quality bar fill | `0 0 6px fill@45%` |
| Active power toggle | `0 0 6px primary@50%` |
| Selected card ring | `0 0 0 2px primary@90%` |
| Bar value text | `text-shadow: 0 1px 2px rgba(0,0,0,0.85)` |

### 10.7 Motion

| Interaction | Transition |
| --- | --- |
| Button hover | `transform: translateY(-1px)`, 120ms ease |
| Quality bar fill | MUI default transition on `width`, `background-color` |
| Toggle | MUI default on `left`, colours, shadow |
| Dim on select | `opacity 160ms ease` |
| Reboot progress | `width 1s linear` |

Ripples are globally disabled (`MuiButtonBase.disableRipple`).

### 10.8 Icons

Library: `@mui/icons-material` only. Sizes are set per component via `"& .MuiSvgIcon-root": { fontSize }` — common values `0.75rem`, `0.8125rem`, `0.9rem`, `1rem`, `1.125rem`, `1.25rem`. Icons are always to the **left** of their label with a `4–6px` gap.

| Meaning | Icon |
| --- | --- |
| Cellular | `SignalCellularAlt` |
| SATCOM | `SatelliteAlt` |
| Radio | `SettingsInputAntenna` / `CellTower` |
| Node / Control Room | `Hub` |
| Camera | `Videocam` |
| Settings | `Settings` |
| Run | `PlayArrow` |
| Reboot | `PowerSettingsNew` |
| Alerts | `Notifications` |
| Expand / collapse | `ExpandMore`, `ExpandLess`, `ChevronRight` |
| Panel collapse | `KeyboardDoubleArrowLeft/Right` |
| Zoom | `Add`, `Remove`, `CenterFocusStrong` |
| Close | `Close` |
| Health | `Thermostat`, `Memory`, `Bolt` |

### 10.9 Scrollbars

`width/height: 5px`, thumb `#9E9E9E` radius `12px`, transparent track. `body { overflow: hidden }` — the app never scrolls as a page; panels scroll internally.

---

## 11. Component specifications

Convention (project memory, mandatory): **one folder per component**, `Component.tsx` + `Component.styles.ts` + `index.ts`; shared components live in `src/components/GLOBAL/`; strict TypeScript props; no hardcoded design values.

### 11.1 Buttons

There is no custom Button component — MUI `Button` is themed globally and specialised per usage.

| Variant | Component | Height / padding | Colours | Hover | Disabled |
| --- | --- | --- | --- | --- | --- |
| Base (theme) | `MuiButton` | `min-height 32px`, radius `4px`, `0.75rem`, uppercase, `0.12em` | inherit | `translateY(-1px)` | `opacity .45`, `cursor:not-allowed` |
| Primary contained | `<Button variant="contained">` | theme default | `primary.main` bg, `#121212` text | `primary.dark` | as base |
| Footer outlined | `SettingsButton` | padding `6px 10px` | `text.secondary`, border `divider` | text `text.primary`, border `primary@50%` | as base |
| Footer outlined, active | `ControlRoomButton active` | padding `6px 10px`, weight 700 | `primary.main` text + border, bg `primary@14%` | border `primary@60%` | as base |
| Map toggle, inactive | `LinksButton` | padding `4px 8px` | bg `paper@80%`, text `text.secondary`, border `divider` | bg `paper@95%` | — |
| Map toggle, active | `LinksButton active` | same | bg `primary.main`, text `contrastText`, border `primary.main` | bg `primary.dark` | — |
| Reboot | `RebootButton` | `min-width 104px`, padding `4px 8px`, `0.75rem/600/0.16em` | text `text.secondary`, border `divider`; running: primary text/border + sweeping `primary@22%` fill | primary tint | `disabled` while running, `aria-busy` |
| Icon button | `MuiIconButton` | radius `4px` | `text.secondary` | `text.primary` | — |
| Camera (full width) | `CameraButton` | padding `2px 3px`, `0.6875rem/0.14em` | `primary@12%` bg, `primary@60%` border | `primary@22%` | — |
| Camera (icon) | `CameraIconButton` | `16×16` | same palette | `primary@24%` | — |

No size scale (`sm/md/lg`) exists. Do not invent one.

### 11.2 `SectionHeader` (GLOBAL)

| Prop | Type | Notes |
| --- | --- | --- |
| `title` | `string` | Rendered uppercase, centred |
| `action` | `ReactNode?` | Absolutely positioned right, vertically centred |
| `emphasis` | `boolean?` | `0.875rem/0.24em/primary.light` instead of `0.75rem/0.2em/text.primary` |

Bar: `panel.header`, padding `7px 12px`, `1px divider` top and bottom.

### 11.3 `QualityBar` / `QualityMeter` (GLOBAL)

`QualityBar` props: `value: number (0-100)`, `disabled?`, `ariaLabel?`, `showValue?`.

- Track: `height 16px`, full width, radius `999px`, bg `background.default`, `1px divider@90%` border, muted → `opacity .55`.
- Fill: status colour by `qualityStatus`, glow `0 0 6px fill@45%`; disabled → `text.secondary` (never red).
- Value: centred inside the track, `0.8125rem/700`, shadowed.
- ARIA: `role="progressbar"` with `aria-valuenow/min/max`.

`QualityMeter` = `QualityBar` with `showValue` always on, wrapped in a flexible row (`min-width: 32px`). This is the **only** approved quality visualisation; the same bar appears in the sidebar, panels, map cards and topology cards.

### 11.4 `SignalBars` (GLOBAL)

5-notch cellular ladder, colour by `qualityStatus`, muted grey when disabled, optional numeric value. Used per channel inside the HALO panel.

### 11.5 `HealthMetrics` (GLOBAL)

Props: `temperature?`, `cpu?`, `voltage?`, `dense?`. Renders outline chips (`transparent` background, `1px` border, radius `4px`, padding `1px 5px`). Good = `divider` border + `text.secondary` text at weight 500; marginal/poor = status colour border and text at weight 700. `dense` sets `0.75rem` and a `4px` gap. Returns `null` when nothing is provided.

### 11.6 `PowerToggle` (GLOBAL)

Track `26×14px`, radius `999px`; thumb `10×10px` at `left: 1px` (off) / `13px` (on). On: border `primary@70%`, bg `primary@30%`, glow. Off: `action.disabledBackground`. `role="switch"` + `aria-checked` + `aria-label`. Always confirms via dialog unless `confirm={false}`; `lastActive` converts the dialog into a blocking warning.

### 11.7 `CollapseButton` (GLOBAL)

Borderless icon button, `0.95rem` icon, `ExpandMore` / `ChevronRight`, `aria-expanded`, `aria-label={"Toggle " + label}`.

### 11.8 `RebootButton` (GLOBAL)

Props: `seconds = 10`, `onStart`, `onComplete`, `onRebootingChange`, `label`. Runs a 1s-tick countdown, disables itself, reports progress through a `::before` fill, and lets the caller dim its own content.

### 11.9 `SeriesCheckbox` (GLOBAL)

Checkbox (`0.9rem` icon, coloured by series) + `2×12px` colour swatch + `0.75rem/0.08em` label. Used by the chart legend and the compare list.

### 11.10 `SidePanel` (GLOBAL)

Props: `side: "left" | "right"`, `width: number`, `header?`, `footer?`, children. Column flex, `panel.surface`, `1px divider` on the inner edge, scrollable body (`flex:1; min-height:0; overflow-y:auto`).

### 11.11 `PlatformCard`

| Prop | Type | Meaning |
| --- | --- | --- |
| `unit` | `PlatformUnit` | Data |
| `variant` | `"overlay" \| "topology"` | Map marker vs logical node |
| `compact` | `boolean` | Fixed summary card |
| `dense` | via width logic | `92px` instead of `104px` |
| `collapsible`, `camera`, `hideTitle`, `selected`, `onSelect` | | |

Widths: overlay `104px` (dense `92px`), topology `176px`. Border: overlay `divider`, topology `status[status]`. Background: overlay `background.default@92%` + `blur(4px)`, topology `background.paper@60%`. Font: overlay `0.625rem`, topology `0.75rem`.

Overlay cards abbreviate names (`PLATFORM 3 → PLT 3`) and range names (`CELLULAR → CEL`); topology cards show **full** names.

Fault badge (`AlertBadge`): `min-width 16px`, `height 16px`, radius `999px`, `background.default` fill (fully opaque), `1px status.poor` border, `status.poor` number, `0.6875rem/700`. Floating variant sits at `top:-8px; right:-6px`.

### 11.12 `MonitoringGraph`

Recharts `LineChart`, `CartesianGrid` dashed `2 3` in `divider`. X: seconds `0–60`, ticks every 10. Y-left: `Mbps 0–25`. Y-right (`withLatency`): `ms 0–200`, width `34px`. Series: Upload = `status.good`, Download = `status.marginal`, Latency = `secondary.main`, Bandwidth = `primary.main` dashed `5 4`. `strokeWidth 1.6`, no dots, `isAnimationActive={false}`. Legend = `SeriesCheckbox` row; unchecking hides a series.

### 11.13 `ViewModeSwitch`

Two-segment control, `TACTICAL` / `LOGICAL`; active segment filled primary. Buttons expose `aria-label` and `aria-pressed`.

### 11.14 `MeshMatrix`

7×7 value grid (`CR`, `P1…P5`, `RLY`), cell background by status, diagonal shows `—`. Above it: three metric checkboxes (`Modulation`, `SNR`, `RSSI`). Below: frequency row (`MESH-A · 2.412 GHz`) and a `GOOD / MARGINAL / POOR` legend.

### 11.15 `ConnectivityWheel`

Titled `MESH LINKS MAP`, expand/collapse control, radial SVG of all nodes with status-coloured chords, and a dot legend. Logical view only, positioned left of the right panel.

---

## 12. Form field specifications

Forms exist **only** in the Settings dialog. All fields are MUI `TextField` (`variant="outlined"`, `size="small"`), styled by `SettingsField`. Theme input styling: radius `4px`, background `rgba(255,255,255,0.09)`, outline `rgba(255,255,255,0.23)`.

Each field is wrapped in `FieldGroup` with a `FieldLabel` above and an optional `FieldHint` below (`0.75rem`, `text.secondary`).

| Tab | Label | Type | Default | Constraints | Hint |
| --- | --- | --- | --- | --- | --- |
| System | Precheck | Button (`Run`, contained, `PlayArrow`) | — | — | "Runs the full communication precheck sequence." |
| Satellite | Satellite | Select | `TELS-1` | `TELS-1`, `TELS-2` | "Active spacecraft used for the SATCOM uplink." |
| Satellite | Beam / Transponder | Text | `KA-04` | none | — |
| Satellite | Symbol rate (Msym/s) | Number | `12` | none | — |
| Satellite | Modulation | Select | `8PSK` | `QPSK`, `8PSK`, `16APSK` | — |
| Radio | Radio frequency (MHz) | Number (controlled) | current value (`2412`) | `min 30`, `max 6000`, `step 0.025` | "Mesh operating frequency shared by all platforms." |
| Radio | Channel bandwidth | Select | `20` | `5`, `10`, `20` MHz | — |
| Radio | Mesh network ID | Text | `MESH-A` | none | — |
| Radio | TX power (dBm) | Number | `27` | none | — |
| Cellular | Preferred network | Select | `5G` | `Auto`, `LTE`, `5G NR` | — |
| Cellular | APN | Text | `fleet.ops` | none | — |
| Cellular | SIM priority | Select | `SIM 1` | `SIM 1..3` | — |
| Cellular | Data cap per SIM (GB) | Number | `50` | none | — |

**Field states:** default / hover / focus / disabled follow MUI defaults for the dark theme. There is **no error state, no helper-error text, no required marker, and no async/loading state** implemented anywhere in the app. If a new feature needs validation, use MUI's `error` + `helperText` on the same `SettingsField` styling — do not invent a new error visual.

Mobile behaviour of fields: `UNKNOWN / NEEDS VERIFICATION` — the dialog is `min(760px, 92vw)` so it fits, but the app shell behind it does not reflow.

---

## 13. Tabular / matrix data specifications

No `<table>` elements exist. Two grid-like structures carry tabular data.

### 13.1 Fleet sidebar lists (`VEHICLES`, `RELAYS`)

| Column | Type | Alignment | Width | Behaviour |
| --- | --- | --- | --- | --- |
| Platform / Relay | Text, weight 700 | Left | `96px` fixed, ellipsis | Row is clickable (`role="button"`) |
| Range | Text (`CELLULAR`, `SATCOM + CELLULAR`, `RADIO`) | Left | `74px` fixed | Static |
| Quality | `QualityMeter` | Fill | `flex: 1`, `min-width 0` | Colour + `%` from `qualityStatus` |

- Header row: `0.75rem`, uppercase, `0.08em`, `text.secondary`.
- Row padding `4px`, negative `4px` inline margin so the selected background bleeds to the panel edge.
- Sorting, filtering, search, pagination, sticky header, multi-select, row menus: **not implemented**.
- Empty state: not implemented (the arrays are non-empty constants).

### 13.2 Link Matrix (`MeshMatrix`)

| Aspect | Spec |
| --- | --- |
| Size | 7 columns × 7 rows plus header row/column |
| Header labels | `CR`, `P1`–`P5`, `RLY` |
| Cell content | Integer margin value; diagonal renders `—` |
| Cell colour | Background by status of the value |
| Alignment | Centred |
| Controls | Metric checkboxes above (`Modulation`, `SNR`, `RSSI`) |
| Footer | Frequency row + colour legend |
| Interactions | None (read-only) |

---

## 14. States catalogue

| Component | Default | Hover | Focus | Active/Selected | Disabled | Loading | Error | Empty |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Sidebar row | transparent | `primary@8%` | browser outline (MUI default) | `primary@12%` + `2px` left primary border | n/a | n/a | n/a | not implemented |
| Map card | `divider` border | tooltip after 600ms; reveals its links when Mesh Links is off | n/a (div) | `0 0 0 2px primary@90%` ring | n/a | n/a | fault badge (count) | n/a |
| Topology card | status-coloured border | tooltip | n/a | ring + other routes dim to `.3` | n/a | n/a | fault badge | n/a |
| `PowerToggle` | grey track | — | MUI focus | primary track + glow | `disabled` prop, e.g. during reboot | n/a | blocked → warning dialog | n/a |
| `RebootButton` | outlined grey | primary tint | MUI focus | — | while running | countdown + `aria-busy` + progress fill | n/a | n/a |
| Mesh Links button | outlined | lighter paper | MUI focus | filled primary, `aria-pressed=true` | n/a | n/a | n/a | n/a |
| Quality bar | status colour | — | — | — | muted grey + `.55` opacity | n/a | n/a | `0%` |
| State chip | status coloured | — | — | — | grey when channel off | n/a | `NO CONN` / `NO SIM` / `UNPLG` | n/a |
| Settings tab | `text.secondary` | MUI hover | MUI focus | `primary.main` + indicator | n/a | n/a | n/a | n/a |
| Dialog buttons | MUI defaults | `translateY(-1px)` | MUI focus | — | `opacity .45` | not implemented | not implemented | n/a |

---

## 15. UX flows

### 15.1 Inspect a platform
1. Operator clicks a `VEHICLES` row (or a map/topology card).
2. Row highlights; the map recentres on the marker (tactical) or the route highlights and others dim (logical).
3. Right panel title becomes the platform name; HALO and RADIO panels re-bind to that platform; the chart gains the latency series; the compare list is hidden.
4. Clicking the same row again, clicking `CONTROL ROOM`, or clicking the logical canvas background returns to the Control Room context.

### 15.2 Expand HALO channels
1. Click the chevron in the HALO panel header (`aria-expanded` flips).
2. Channel rows render, each on exactly one line.
3. Clicking a channel name with RF data expands `SINR / RSSI / RSRP`.
4. Collapse restores the summary rows.

### 15.3 Power a channel off
1. Operator clicks a channel `PowerToggle`.
2. Confirmation dialog opens (`CONFIRM POWER CHANGE`).
3. `Confirm` → the channel is switched off locally: bars mute, state chip becomes `NO CONN`, rate becomes `—`.
4. If the channel is the last active range, the dialog opens as `ACTION BLOCKED` with a warning alert and only a `Close` action; nothing changes.

### 15.4 Reboot the HALO unit
1. Expand the panel and click `REBOOT`.
2. The button disables, shows `REBOOTING 10s…1s` and a sweeping fill; all panel content dims; toggles are disabled.
3. At zero the panel restores. No confirmation dialog, no toast, no server call.

### 15.5 Change a setting
1. Click `SETTINGS`.
2. Dialog opens on the `System` tab; the operator switches tabs and edits fields.
3. `Apply` parses the radio frequency, lifts it to the panel state and closes. `Cancel`, ESC or backdrop click close without applying.
4. No success or error feedback exists.

### 15.6 Open a camera feed
1. Click the camera button on a platform card.
2. A portal window opens over a dark backdrop.
3. Quality can be switched `H/M/L` via a menu.
4. Backdrop click or the close button dismisses it.

### 15.7 Failure flow
Not implemented anywhere. There is no request layer, so there is no retry, no error banner, no toast. Any new feature that talks to a backend must define this flow explicitly (see §21).

---

## 16. Loading, empty and error states

| Screen / area | Loading | Empty | Error |
| --- | --- | --- | --- |
| Tactical map | none — data is static; the raster loads with the page | not possible | none |
| Logical topology | none | not possible | none |
| Sidebar lists | none | not implemented | none |
| Right panel | none | shows the Control Room context when nothing is selected (this is the closest thing to an empty state) | none |
| Monitoring chart | Recharts renders once the container has size; there is no skeleton, so the chart area can flash empty on first paint | none | none |
| Settings | none | none | none |
| Reboot | the only real loading pattern: dim content + disabled controls + countdown | — | — |

**Rule for new features:** implement all three states using the existing vocabulary — dim + disable for loading (as `RebootButton`/`DimWrap` do), a centred `0.75rem text.secondary` message inside the owning panel for empty, and an outlined MUI `Alert severity="error"` (matching the blocked-power dialog) for errors.

---

## 17. Responsive behaviour

MUI default breakpoints are in force (`xs 0, sm 600, md 900, lg 1200, xl 1536`), but **no component queries them**. There is not a single `theme.breakpoints` usage in `src/components`.

| Width | Actual behaviour |
| --- | --- |
| ≥ ~1400px | Intended experience. Viewport ≈ `width − 636px`. |
| 1200–1400px | Works; map viewport narrows; logical diagram scales down to its `min-width: 720px` floor. |
| 1024px (`13-tablet-1024.png`) | Still three fixed columns; the viewport is only ~388px wide; the logical diagram hits its minimum and can clip. |
| < 900px | Broken: the three columns do not fit, content is cut off. |
| 390px (`14-mobile-390.png`) | Broken: only the sidebar is usable; the viewport and right panel are pushed off screen. There is no stacking, no drawer conversion, no bottom navigation. |

Manual mitigations that exist: the left sidebar can be collapsed to a `32px` rail, which returns `264px` to the viewport.

`UNKNOWN / NEEDS VERIFICATION`: no intended mobile design exists. Do not invent one silently — if a feature must work on mobile, agree the layout first.

---

## 18. Accessibility

Implemented today:

- Landmarks: `<main>` shell, `<aside>` side panels, `<header>`/`<footer>` inside panels, one `<h1>` (sidebar title).
- `aria-label` on icon-only controls: `Collapse panel`, `Expand panel`, `Close settings`, `Notifications (n)`, `Close camera window`, `Feed quality`, zoom buttons, `Select PLATFORM n`, `Center map on RELAY`.
- `aria-expanded` on collapse buttons and RF-metric channel buttons.
- `aria-pressed` on `MESH LINKS` and the view-mode segments.
- `role="switch"` + `aria-checked` on power toggles; `role="progressbar"` + `aria-valuenow/min/max` on quality bars; `aria-busy` on the rebooting button.
- Tooltips carry the full text for every abbreviation (state chips, short names, health chips).
- Dialogs use MUI focus trapping; the settings dialog is `aria-labelledby`.

Gaps (fix when you touch these areas):

- Sidebar and card rows are `div`s with `role="button"` and no `tabIndex`/key handler → not keyboard reachable.
- The camera window does not close on ESC and is not a focus-trapped MUI dialog.
- SVG topology edges and map links have no text alternative; status is conveyed by colour alone in the mesh matrix.
- Contrast: `text.secondary` at `0.625rem` inside badges is below comfortable reading size; verify against WCAG AA before reusing at that size.
- No skip link, no visible custom focus ring (MUI default only).

---

## 19. Code mapping

| UI area | File | Key exports |
| --- | --- | --- |
| Route + state owner | `src/routes/index.tsx` | `Route`, `MonitorPage` |
| Root document, fonts, theme provider | `src/routes/__root.tsx` | — |
| Theme tokens | `src/theme/tacticalTheme.ts` | `tacticalTheme`, `statusColor`, `LinkStatus` |
| Status thresholds | `src/lib/linkStatus.ts` | `qualityStatus`, `rateStatus`, `temperatureStatus`, `cpuStatus`, `voltageStatus`, `parseRate` |
| Latency estimation | `src/lib/telemetry.ts` | `estimateLatency` |
| Shell | `src/components/MonitorLayout/` | `MonitorLayout` |
| Center viewport + top-right bar | `src/components/MonitorViewport/` | `MonitorViewport`, `LinksButton`, `TopRightBar` |
| Left panel | `src/components/FleetSidebar/` | `FleetSidebar` and its cell styles |
| Right panel | `src/components/ControlRoomPanel/` | `ControlRoomPanel`, `PanelFooter`, `ControlRoomButton`, `SettingsButton` |
| HALO panel | `src/components/ModemPanel/` | `ModemPanel`, `StateChip`, `DimWrap` |
| Radio panel | `src/components/RadioPanel/` | `RadioPanel` |
| Tactical map | `src/components/TacticalMap/` | `TacticalMap`, `mapGeometry.ts` |
| Logical topology | `src/components/LogicalTopology/` | `LogicalTopology`, `LayerStack`, `CommandNode`, `HopNode`, `RouterModule`, `PlatformRow`, `Legend` |
| Platform card | `src/components/PlatformCard/` | `PlatformCard`, `AlertBadge`, `CardRoot` |
| Mesh matrix | `src/components/MeshMatrix/` | `MeshMatrix` |
| Mesh links wheel | `src/components/ConnectivityWheel/` | `ConnectivityWheel` |
| Chart | `src/components/MonitoringGraph/` | `MonitoringGraph`, `chartAxisStyles` |
| Compare window | `src/components/ModemCompareWindow/` | `ModemCompareWindow` |
| Settings | `src/components/SettingsDialog/` | `SettingsDialog`, `SettingsField`, `FieldGroup`, `FieldLabel`, `FieldHint` |
| Camera | `src/components/CameraWindow/` | `CameraWindow` |
| Alerts | `src/components/NotificationTray/` | `NotificationTray` |
| View switch | `src/components/ViewModeSwitch/` | `ViewModeSwitch` |
| Coordinates | `src/components/CoordinateDialog/` | `CoordinateDialog` |
| Shared primitives | `src/components/GLOBAL/*` | `SectionHeader`, `SidePanel`, `QualityBar`, `QualityMeter`, `SignalBars`, `HealthMetrics`, `PowerToggle`, `CollapseButton`, `RebootButton`, `SeriesCheckbox`, `StatusIndicator` |
| Hooks | `src/hooks/` | `useMapViewport`, `useMapDrag`, `useToggleList` |
| Data | `src/data/` | `network.ts` (platforms, relays, links, `MAX_BANDWIDTH_MBPS = 22`, positions), `modems.ts`, `notifications.ts` |
| Types | `src/types/network.ts` | `PlatformUnit`, `RelayUnit`, `ModemAsset`, `RadioAsset`, `LinkKind`, `ViewMode`, … |

There are **no Tailwind classes and no CSS files** used by components; `src/styles.css` exists for the base document only.

---

## 20. Design inconsistencies / existing UX debt

| # | Issue | Where | Recommended standard |
| --- | --- | --- | --- |
| 1 | View mode is not in the URL | `src/routes/index.tsx` | Move `mode` to a search param (`?view=logical`) so states are shareable |
| 2 | `ControlRoomDrawer` is implemented but never mounted | `src/components/ControlRoomDrawer/` | Delete it, or adopt it — right now it is a second, competing pattern for the same job the right panel already does |
| 3 | Abbreviations differ by surface: map cards say `PLT 3` / `CEL`, topology cards say `PLATFORM 3` / `CELLULAR` | `PlatformCard.tsx` | Keep as-is (space-driven), but never abbreviate without a tooltip carrying the full text |
| 4 | Card title CSS still forces `nowrap + ellipsis` while the topology parent allows wrapping | `PlatformCard.styles.ts` | If a topology label ever clips, fix it on `CardTitle` per variant, not with a new component |
| 5 | Two different "collapse" affordances: bordered icon button (sidebar) vs borderless chevron (panels) | `FleetSidebar.styles.ts`, `GLOBAL/CollapseButton` | Bordered button for whole panels; borderless chevron for content sections |
| 6 | No loading / empty / error vocabulary anywhere | whole app | Adopt the rules in §16 for anything new |
| 7 | Clickable `div`s are not keyboard operable | sidebar rows, cards | Convert to `ButtonBase` when you next touch them |
| 8 | Camera window ignores ESC and is not focus-trapped | `CameraWindow.tsx` | Rebuild on MUI `Dialog` when touched |
| 9 | No responsive behaviour below ~1200px | all screens | Decide the mobile story before adding mobile-facing features |
| 10 | Chart can flash an empty plot area on first paint | `MonitoringGraph` | Reserve the canvas height (already fixed) and avoid animation; add a skeleton if a real data source appears |
| 11 | `MeterScore` and `SatelliteName` styled exports are unused leftovers | `GLOBAL/QualityMeter.styles.ts`, `FleetSidebar.styles.ts` | Remove during the next cleanup; do not build on them |
| 12 | Legend/edge crowding at the bottom of the logical view at short viewport heights | `LogicalTopology.styles.ts` | Keep the reserved bottom padding (`spacing(13)`) when changing that layout |

---

## 21. Rules for new features

**Architecture**
1. One folder per component: `Component.tsx`, `Component.styles.ts`, `index.ts`. No exceptions.
2. All styling through MUI `styled()` reading `theme`. No inline hex, no `sx` with literal colours, no CSS files, no Tailwind.
3. Shared/reused pieces go in `src/components/GLOBAL/`; feature-specific pieces stay in their own folder.
4. Strict, explicit TypeScript prop interfaces, exported next to the component. Document non-obvious props with a short JSDoc line.
5. Business types live in `src/types/network.ts`; status logic lives in `src/lib/linkStatus.ts`. Never re-derive thresholds locally.

**Visual**
6. Use existing tokens only (§10). New colours require an explicit decision; new radii and spacing steps are not allowed.
7. Quality is always shown with `QualityBar`/`QualityMeter`, never with a new bar, gauge or ring.
8. Health is always shown with `HealthMetrics` chips.
9. Status colour comes from `theme.palette.status[...]` via the `linkStatus` helpers.
10. Every section title uses `SectionHeader`; every side panel uses `SidePanel`.
11. Buttons: reuse MUI `Button` with the existing styled wrappers. Do not create a new button component.
12. Icons: `@mui/icons-material` only, left of the label, sized through `& .MuiSvgIcon-root`.
13. Text is uppercase for titles, labels and buttons; numeric readouts use tabular numerals.
14. Minimum font size `0.625rem`, and only for badges; body text is `0.75rem` or larger.

**Behaviour**
15. Any destructive or state-changing hardware action goes through a confirmation dialog matching `PowerToggle`'s.
16. Long operations dim their own region and disable their controls (the `DimWrap` + `RebootButton` pattern). Never block the whole app.
17. Selecting an entity updates the right panel; it never opens a new page.
18. Overlays: use MUI `Dialog` for modal work, MUI `Popover` for anchored lists, the portal window pattern only for media.
19. Abbreviate only when space forces it, and always provide the full text in a `title`/`Tooltip`.
20. Implement loading, empty and error states per §16 for anything that can fail.
21. Preserve the existing keyboard and ARIA affordances listed in §18; add `aria-label` to every icon-only control you introduce.

**Forbidden**
- New global layouts or a second navigation model.
- A second theme, light mode, or a colour outside the palette.
- Toast libraries, new chart libraries, new icon sets, CSS-in-JS other than MUI.
- Reintroducing the words "MODEM" or "ROUTER" in UI copy — the product term is **HALO**.
- Removing the `CONTROL ROOM` node's transparent background in the logical view (it would hide the connector lines).

---

## 22. Feature implementation template

Copy this block into the task description for every new feature.

```markdown
# New Feature Specification

## Feature name


## Entry point
(Which existing control opens it: sidebar row / right-panel button / map marker / settings tab)

## Route
(`/` with new local state, or a new route file under src/routes/)

## User flow
1.
2.
3.

## Screen layout
(Which of the three columns changes; block order top-to-bottom; widths in px or %)

## Components reused
- SectionHeader / SidePanel / QualityMeter / HealthMetrics / PowerToggle / RebootButton / SeriesCheckbox / PlatformCard / ...

## New components required
(name, folder, why an existing one cannot be reused)

## Fields
| Label | Type | Default | Required | Validation | Error message | Hint |
|---|---|---|---|---|---|---|

## Buttons
| Label | Variant | Icon | Action | Confirmation? |
|---|---|---|---|---|

## Data
(source module, types added to src/types/network.ts, thresholds added to src/lib/linkStatus.ts)

## States
- Default:
- Hover:
- Focus:
- Selected:
- Disabled:
- Loading:
- Empty:
- Error:

## Responsive behaviour
(Desktop ≥1400 / 1024 / below 900 — state explicitly if unsupported)

## Accessibility
(aria-label, role, aria-expanded/pressed, keyboard path, focus order)

## Tokens used
(colour, spacing, radius, typography — token names only, no literals)
```

---

## 23. Developer handoff — Claude Code

### The design language in one paragraph
A dark, high-density tactical console. One dark theme (`#121212` panels on a `#303030` app background), light-blue primary `#90CAF9`, Exo typography with wide uppercase letter-spacing on every title and button, 4px spacing grid, 4px radius on almost everything, pill-shaped quality bars, outline-only status chips, and a strict three-colour status scale (green `#66BB6A` / orange `#FFA726` / red `#F44336`). Nothing is decorative: every colour on screen encodes link, health or selection state.

### Components you must reuse
`SidePanel`, `SectionHeader`, `QualityBar`, `QualityMeter`, `SignalBars`, `HealthMetrics`, `PowerToggle`, `CollapseButton`, `RebootButton`, `SeriesCheckbox`, `PlatformCard`, `MonitoringGraph`, `ViewModeSwitch`, plus MUI `Button`, `Dialog`, `Popover`, `Menu`, `Tabs`, `TextField`, `Tooltip`, `Badge`, `Alert`.

### Tokens you must use
`theme.palette.*` (including `status`, `panel`, `interactive`), `theme.spacing(n)` (4px base), `theme.shape.borderRadius` (4), the typography scale in §10.3, and the `linkStatus` helpers for every threshold decision.

### Patterns you must not break
1. Three fixed columns; the centre is the only free canvas.
2. Selection drives the right panel; no page navigation.
3. Confirmation before any power change; a blocked action explains itself instead of failing silently.
4. Long operations dim their own region only.
5. HALO terminology.
6. One quality visualisation, one health chip set, one status colour scale.
7. Abbreviation always accompanied by a tooltip.

### How to build a new feature
1. Read §19 and open the closest existing component.
2. Fill in the §22 template.
3. Create the folder pair (`.tsx` + `.styles.ts` + `index.ts`).
4. Wire it into `src/routes/index.tsx` state or into the panel that owns the context.
5. Implement all states from §14 and §16.
6. Verify against §21, then run the checklist below.

### Desktop checks
- 1600×1000 and 1280×800: no clipping, no horizontal scrollbar, panels scroll internally.
- Right panel footer stays visible while the panel body scrolls.
- Selection ring, dimming and tooltips behave as in §14.

### Mobile checks
- Understand that below ~1200px the shell is already broken (§17). Do not claim mobile support you have not implemented; if the feature must be mobile-usable, agree the layout first.

### States that must always exist
Default, hover, focus, selected, disabled, loading, empty, error — for every interactive element you add.

### New components: allowed vs forbidden
- **Allowed:** a feature-specific composition of existing primitives (e.g. a new panel section, a new dialog body, a new topology node type).
- **Forbidden:** a new button, input, bar, chip, badge, tooltip, modal shell, panel shell or section header.

---

### Checklist

**Before implementation**
- [ ] Reviewed existing components in `src/components` and `src/components/GLOBAL`
- [ ] Reviewed design tokens in `src/theme/tacticalTheme.ts`
- [ ] Reviewed the relevant screens and screenshots in `screenshots-v2/`
- [ ] Reviewed responsive limits (§17) and known UX debt (§20)

**During implementation**
- [ ] Reused existing components instead of creating new ones
- [ ] Used only theme tokens — no literal colours, radii or font sizes outside the scale
- [ ] Component folder contains `.tsx`, `.styles.ts`, `index.ts`
- [ ] Implemented loading state (dim + disable pattern)
- [ ] Implemented empty state
- [ ] Implemented error state (outlined `Alert`)
- [ ] Implemented confirmation for state-changing actions
- [ ] Added `aria-label` / `role` / `aria-expanded` / `aria-pressed` as applicable
- [ ] Every abbreviation has a tooltip with the full text

**Before delivery**
- [ ] Desktop matches the documented layout and spacing
- [ ] Behaviour below 1200px is either handled or explicitly declared unsupported
- [ ] All states implemented and visually verified
- [ ] No unnecessary new components
- [ ] Existing UI patterns preserved (three columns, right-panel selection, HALO wording)
- [ ] `bunx tsgo --noEmit` passes and the preview shows no console errors
