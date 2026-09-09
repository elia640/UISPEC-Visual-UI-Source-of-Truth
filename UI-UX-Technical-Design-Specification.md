# UI/UX Technical Design Specification
## Civil Network Monitoring System — Operator Console

**Document type:** Reverse-engineered design specification (source of truth for new features)
**Audience:** Claude Code / software developers adding features to this system
**Status of values:** Every number in this document is read from the source code or measured on the running application at 1600×950. Values that cannot be determined are marked `UNKNOWN / NEEDS VERIFICATION`.
**Scope rule:** This document describes the system as it is today. It is not a redesign proposal.
**Companion document:** `UISPEC.md` is the implementation-oriented visual/UI source of truth (tokens, exact measurements, component contracts, forbidden patterns, feature template, checklist). Where the two documents differ, `UISPEC.md` wins.

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Screen Inventory](#2-screen-inventory)
3. [Screen Specifications](#3-screen-specifications)
4. [Screenshots](#4-screenshots)
5. [Component Specifications](#5-component-specifications)
6. [Form Fields](#6-form-fields)
7. [Tables](#7-tables)
8. [Modals, Dialogs and Drawers](#8-modals-dialogs-and-drawers)
9. [Responsive Design](#9-responsive-design)
10. [Design System](#10-design-system)
11. [States](#11-states)
12. [UX Flows](#12-ux-flows)
13. [Empty / Loading / Error States](#13-empty--loading--error-states)
14. [Accessibility](#14-accessibility)
15. [Component Architecture](#15-component-architecture)
16. [Code Mapping](#16-code-mapping)
17. [Rules for New Features](#17-rules-for-new-features)
18. [Feature Implementation Template](#18-feature-implementation-template)
19. [Design Inconsistencies / Existing UX Debt](#19-design-inconsistencies--existing-ux-debt)
20. [Developer Handoff — Claude Code](#20-developer-handoff--claude-code)

---

## 1. System Overview

### 1.1 Purpose

A single-screen operator console for monitoring the communication network of a remotely operated / autonomous vehicle fleet. The operator sees where each platform is, which communication ranges (CELLULAR / SATCOM / RADIO) it uses, the quality and throughput of each link, and can power individual links, modems and channels on or off, reboot modems, open camera feeds and configure RF parameters.

### 1.2 Technical stack (verified)

| Concern | Implementation |
|---|---|
| Framework | React 19 + TanStack Start v1 (SSR), Vite 7 |
| Router | TanStack Router (file-based, `src/routes/`) |
| UI library | **MUI v7** (`@mui/material`, `@mui/icons-material`) |
| Styling | MUI `styled()` + theme only. **No Tailwind classes are used in application components.** Tailwind v4 is present in `src/styles.css` for base/reset only. |
| Charts | `recharts` (`LineChart`) |
| State | Local React `useState` lifted to `src/routes/index.tsx`. No global store, no server state, no data fetching. |
| Data | Static fixtures in `src/data/network.ts`, `src/data/modems.ts`, `src/data/notifications.ts` |
| Theme file | `src/theme/tacticalTheme.ts` (single source of design tokens) |

### 1.3 Architecture conventions (mandatory, already enforced project-wide)

- One folder per component: `ComponentName/ComponentName.tsx`, `ComponentName/ComponentName.styles.ts`, `ComponentName/index.ts`.
- All visual values come from the MUI theme (`theme.palette`, `theme.spacing`, `theme.shape`, `theme.typography`). Hard-coded hex colors in components are forbidden.
- Shared/reusable components live in `src/components/GLOBAL/`.
- Props interfaces are exported and named `<ComponentName>Props`. TypeScript is strict (`exactOptionalPropertyTypes` behaviour is visible in the codebase — optional callbacks are spread conditionally, e.g. `{...(onSelectVehicle ? { onSelectVehicle } : {})}`).

---

## 2. Screen Inventory

### 2.1 Routes

| Route | File | Purpose |
|---|---|---|
| `/` | `src/routes/index.tsx` | The entire application — the operator console |
| (root layout) | `src/routes/__root.tsx` | HTML shell, font links, `ThemeProvider`, `CssBaseline` |

There is **one route only**. The application is a persistent single-screen console; "screens" below are *modes and regions* of that route, not separate URLs.

### 2.2 Screens / modes

| # | Screen | How reached | Key content |
|---|---|---|---|
| S1 | Tactical View (default) | `mode = "tactical"` — Tactical button in right panel footer | Satellite map, platform markers, links, legend, zoom controls |
| S2 | Logical View | Logical button in right panel footer | Layered topology graph, Connectivity Map window |
| S3 | Control Room Panel (right column) | Always visible | Modem panel, Radio panel, monitoring graph, compare list, footer controls |
| S4 | Fleet Sidebar (left column) | Always visible; collapsible to a 32 px rail | Vehicles list, Relays list, Link Matrix |
| S5 | Vehicle context of S3 | Select a platform in S4 or on the map | Same panel, populated with that platform's modem/radio/graph |

### 2.3 Overlay surfaces

| Surface | Type | Trigger |
|---|---|---|
| Settings | Modal dialog, tabbed (System / Satellite / Radio / Cellular) | `SETTINGS` button, right panel footer |
| Control Room Parameters | Right-side Drawer (360 px) | Click the CONTROL ROOM node in Logical view |
| Camera feed | Fixed-position modal window with backdrop | `CAMERA` icon button on a map platform card |
| Confirm power change | MUI Dialog (`maxWidth="xs"`, fullWidth) | Any `PowerToggle` click |
| Action blocked (last link) | Same dialog, warning variant | Toggling off the last active link |
| Coordinate entry | MUI Dialog | `CoordinateDialog` (map coordinate input) |
| Modem compare window | Draggable floating window, 380 px | Checkbox in "Compare Modem Monitoring" |
| Notification tray | MUI Popover/menu with badge | Bell icon, top-right of viewport |
| Connectivity Map | Fixed draggable panel (190 px collapsed / 420 px expanded) | Always present in Logical view |

### 2.4 UI patterns present / absent

| Pattern | Present? | Notes |
|---|---|---|
| Tabs | Yes | Settings dialog only (4 tabs) |
| Tables | Yes | Link Matrix (`<table>`); Vehicles/Relays are flex-based pseudo-tables |
| Dropdowns / Select | Yes | Settings dialog (`TextField select`) |
| Checkboxes | Yes | `SeriesCheckbox` (chart series, compare list, matrix metric modes) |
| Toggles | Yes | `PowerToggle` (26×14 px custom switch) |
| Modals / Drawers | Yes | See 2.3 |
| Confirmation dialogs | Yes | Every power state change |
| Toasts / snackbars | **No** | No toast system is installed |
| Search | **No** | |
| Filters | Partial | Matrix metric checkboxes; chart series checkboxes |
| Pagination | **No** | All lists are fixed length (5 platforms, 1 relay) |
| Empty states | **No** | No dataset can be empty — data is static |
| Loading states | **Only** the REBOOT countdown | No spinners, no skeletons |
| Error states | **No** | No async operations exist |

---

## 3. Screen Specifications

### 3.0 Global layout shell

`MonitorLayout` (`src/components/MonitorLayout/MonitorLayout.styles.ts`):

```
display: flex; height: 100vh; width: 100%;
overflow: hidden; background: palette.background.default (#303030)
```

Three columns, left to right (LTR; the UI language is English):

| Column | Component | Width | Behaviour |
|---|---|---|---|
| Left | `FleetSidebar` → `SidePanel side="left"` | **296 px** fixed (`flexShrink: 0`); **32 px** when collapsed | Vertically scrollable body |
| Center | `MonitorViewport` | `flex: 1; minWidth: 0` | Map or topology, position relative |
| Right | `ControlRoomPanel` → `SidePanel side="right"` | **340 px** fixed | Header pinned, body scrolls, footer pinned |

`SidePanel` internals: root is `display:flex; flexDirection:column; height:100%; overflow:hidden; background: palette.panel.surface (#121212)`; body is `flex:1; minHeight:0; overflowY:auto`. Left panel has `borderRight: 1px solid divider`, right panel `borderLeft`.

---

### 3.1 S1 — Tactical View

**Screen name:** Tactical View
**Route:** `/` with `mode = "tactical"`
**Component:** `src/components/TacticalMap/TacticalMap.tsx`
**Purpose:** Geographic situational awareness — platform positions, live link colors, per-platform quick data.
**Users:** Fleet communication operator.

#### Layout (absolute layers, bottom → top)

1. `MapRoot` — `position:absolute; inset:0; overflow:hidden`, holds the transformed map surface.
2. `MapImage` — `src/assets/map-satellite.jpg`, `width/height:100%`, `objectFit: cover`, `userSelect: none`.
3. `OverlaySvg` — `position:absolute; inset:0; width/height:100%; pointerEvents:none` — draws all link polylines.
4. `AnchoredPoint` elements — `position:absolute`, positioned by percentage `left/top` from `unit.x` / `unit.y` in `src/data/network.ts`, `transform: translate(-50%, -50%)`.
5. Fixed chrome: coordinate chip (top-left), viewport title (top-center), Mesh Links + notification tray (top-right), legend (bottom-left, `left/bottom: spacing(3) = 12px`), zoom controls (bottom-right).

#### Map interaction (`src/hooks/useMapViewport.ts`, `useMapDrag.ts`)

- **Pan:** left mouse button hold + drag on empty map surface only. Right/middle button does not pan. Drag on a card/marker does not pan.
- **Zoom:** mouse wheel, and `+` / `−` / reset buttons bottom-right.
- Markers and cards are anchored in map space and scale with the transform.

#### Node types

| Node | Shape | Size | Notes |
|---|---|---|---|
| Platform | `NodeBadge shape="square"` | 30 × 30 px, radius 4 | Truck icon, 1 px status-colored border |
| Control Room | `NodeBadge shape="circle"` | 36 × 36 px | Draggable (left button), `PingRing` animation, `NodeLabel` "CONTROL ROOM" below |
| Relay | Square badge + `RELAY` semantics | 30 × 30 px | Draggable anywhere on the map |
| Satellite | Icon + `NodeLabel` "TELS-1" | — | Static at `SATELLITE_POSITION = {x:84, y:12}` |
| Off-screen Control Room | Direction arrow at map edge, abbreviation **CR** | — | Shown only when the node is outside the viewport |

`NodeLabel`: `padding: 2px 6px`, `fontSize: 0.75rem (12px)`, `letterSpacing: 0.14em`, `color: primary.main`, `border: 1px solid divider`, radius 4.

#### Platform card (compact overlay) — `PlatformCard variant="overlay"`

Width **104 px** (`dense` = 92 px). Font size 0.625rem (10 px). Border 1 px, radius 4, status/selection-colored.

Rows, top to bottom:
1. `CardHeader` — `CardTitle` (abbreviated name, e.g. `PLT 1`, `fontWeight:800`, `fontSize:1.05em`, ellipsis) + `KindBadges` (e.g. `CEL`, `SAT`, `RAD`; 10 px, grey, 1 px border, `padding: 0 3px`).
2. `CardSection` — `SignalBars` + percentage + `CAMERA` icon button.
3. Alert badge (red circular count) when the platform has notifications.

Cards do **not** expand in the tactical view. All deeper data lives in the right panel.

#### Overlapping markers

When two markers overlap, they fan out; the hovered or selected marker is raised in z-order.

#### Link rendering rules (verified in `TacticalMap.tsx`)

| Case | Style |
|---|---|
| Platform actively transmitting on RADIO → Control Room | Solid polyline, status color |
| Platform on another range but radio-reachable | Dashed polyline, status color |
| Non-radio platform | **No line to Control Room** |
| Relay → served platforms | Line drawn from relay position |
| Colors | `status.good #66BB6A`, `status.marginal #FFA726`, `status.poor #F44336` |
| `Mesh Links` off | All mesh polylines hidden; hovering a marker reveals only that marker's links |

#### Fixed chrome specs

- **Coordinate chip** (`InfoChip`): `padding: 4px 6px`, 12 px text, `letterSpacing: 0.16em`, grey text, primary icon 0.8rem.
- **Viewport title** (`ViewportTitle`): absolute, `left:50%; top:12px; transform: translateX(-50%)`, `fontSize 0.875rem (14px)`, `fontWeight 700`, `letterSpacing 0.2em`, `pointerEvents:none`.
- **Top-right bar** (`TopRightBar`): `right:12px; top:12px`, flex, `gap: 6px`, `zIndex: 5`.
- **Mesh Links button** (`LinksButton`): `padding: 4px 8px`, 12 px, 1 px border; active = `primary.main` background + `primary.contrastText`; inactive = `alpha(background.paper, 0.8)` + secondary text. `aria-pressed` reflects state.
- **Legend** (`LegendBox`): bottom-left, 12 px text, four rows — good / marginal / poor radio link, "RADIO CONNECTIVITY ONLY".

---

### 3.2 S2 — Logical View

**Component:** `src/components/LogicalTopology/LogicalTopology.tsx`
**Purpose:** Show the routing hierarchy — which platform reaches the Control Room through which intermediate node.

#### Three-column layered graph

| Column | Caption | Content |
|---|---|---|
| Left | `PLATFORMS` | Two clusters: `DIRECT · ROUTER GROUP` (primary blue accent) and `RELAY GROUP` (warning orange accent). Each member is a `PlatformCard variant="topology"` (**220 px** wide, 0.6875rem/11 px base) with an `EdgeLabel` above it, e.g. `CELLULAR → ROUTER`. |
| Middle | `NETWORK NODES` | `ROUTER` node (blue, caption "N DIRECT PLATFORMS") and `RELAY` node (orange dashed, caption "N RELAYED PLATFORMS"). |
| Right | `CONTROL ROOM` | `TELS-1 SATELLITE` node, then the CONTROL ROOM container holding the internal `J8` modem module (`ACTIVE`). |

#### Edge routing

Edges are measured at runtime (`useLayoutEffect` + `ResizeObserver` + window resize) from real DOM rects and drawn as **orthogonal elbows**: `M x1 y1 H midX V y2 H x2`.

- Vehicle → hop edge color = `palette.status[unit.status]`.
- Hop → J8 edge color = blue for Router, orange for Relay.
- Relay pathway edges are dashed (`strokeDasharray "5 4"`); router pathway edges are solid.
- All incoming edges terminate at the **J8 modem element**, not at the Control Room box.
- Relay-served vehicles have **no** direct edge to Router or Control Room.

#### Path highlighting

Hover or selection of a vehicle sets `activeRoute`. Edges on that route: `strokeWidth 2.4`, `opacity 0.95`. All other edges: `opacity 0.18`. Unrelated cards and hop nodes are dimmed. Clicking a card toggles selection and drives the right panel.

#### Legend (bottom)

`DIRECT → ROUTER` (solid blue), `VIA RELAY` (dashed orange), `ACTIVE` (green), `WARNING` (orange), `DEGRADED` (red).

#### Connectivity Map (`ConnectivityWheel`)

- Logical view only. `position: fixed`, bottom-right, `zIndex: 3`.
- Width **190 px** collapsed / **420 px** expanded, animated via `theme.transitions.create("width")`.
- Header: title `CONNECTIVITY MAP` (12 px, `letterSpacing 0.14em`, primary) + expand/collapse icon button (2 px padding, 1 px border, 0.8rem icon).
- Body: SVG circle layout — nodes at `RADIUS 34` around `CENTER 50` in a 100×100 viewBox; node labels `CR`, `P1`–`P5`, `RLY`. Edge color from link margin: `>= 16 good`, `>= 10 marginal`, else `poor`.
- Draggable with the left mouse button by its header.

---

### 3.3 S3 — Control Room Panel (right column)

**Component:** `src/components/ControlRoomPanel/ControlRoomPanel.tsx`, width **340 px**.

Vertical order:

1. **`SectionHeader emphasis`** — title = selected platform label, or `CONTROL ROOM` when nothing is selected.
   `SectionHeaderBar`: `padding: 7px 12px`, background `panel.header #212121`, 1 px top and bottom borders, content centered.
   `emphasis` title: 0.875rem (14 px), `fontWeight 800`, `letterSpacing 0.24em`, `color primary.light #BBDEFB`. Non-emphasis: 0.75rem (12 px), `letterSpacing 0.2em`, `color text.primary`.
2. **`PanelStack`** — `padding: 10px 12px`, `gap: 10px`, containing:
   - `ModemPanel` (cellular modem, collapsible)
   - `RadioPanel` (radio asset, collapsible, with its own power toggle)
3. **`SectionHeader`** — `Control Room Monitoring` or `Modem Monitoring` (vehicle selected).
4. **`MonitoringGraph`**.
5. **Compare block** (Control Room context only): caption `COMPARE MODEM MONITORING` (12 px, `fontWeight 700`, `letterSpacing 0.14em`, secondary text) plus `CompareRow` — a bordered one-per-row list of `SeriesCheckbox` items (`padding: 5px 8px`, 1 px separator between rows).
6. **`PanelFooter`** — pinned, `padding: 10px 12px`, top border, background `panel.header`, `flexWrap: wrap`, `gap: 6px`:
   - Row 1 (`FooterRow`, `flexBasis:100%`): `ViewModeSwitch` — MUI ToggleButtonGroup, exclusive, values `Tactical` / `Logical`.
   - Row 2: `CONTROL ROOM` outlined button (active when no vehicle is selected: `primary.main` text/border, `alpha(primary,0.14)` fill), spacer, `SETTINGS` outlined button.

Selecting a platform anywhere (sidebar, map card, topology card) swaps the header title, modem, radio and graph data (`vehicleModem(id)`, `vehicleRadio(id)`, `monitoringSamples(seed)`) and adds the Latency series with a right-hand `ms` axis. Compare checkboxes are hidden in vehicle context.

---

### 3.4 S4 — Fleet Sidebar (left column)

**Component:** `src/components/FleetSidebar/FleetSidebar.tsx`, width **296 px**.

- **Header** (`SidebarHeader`): `padding: 12px`, background `panel.header`, `borderBottom: 2px solid alpha(primary, 0.5)`. Title `COMMS NETWORK MONITOR` rendered as `h1`, 0.875rem (14 px), `fontWeight 700`, `letterSpacing 0.2em`, `color primary.main`. Collapse icon button on the right: 1 px border, radius 4, `padding 4px`, icon 0.9rem, `aria-label="Collapse panel"`.
- **Collapsed rail** (`CollapsedRail`): 32 px wide, full height, `ButtonBase`, chevron icon + vertical title (`writingMode: vertical-rl`, 12 px, `letterSpacing 0.2em`, primary), `aria-label="Expand panel"`.
- **Sections** (each preceded by a centered `SectionHeader`): `Vehicles`, `Relays`, `Link Matrix`.
- **List rows** (`ListRow`): `padding: 4px`, `marginInline: -4px`, `gap: 8px`, `borderLeft: 2px solid` (primary when selected, transparent otherwise), selected background `alpha(primary, 0.12)`, hover `alpha(primary, 0.08)` / `0.18` when selected. `role="button"`, `aria-label="Select PLATFORM n"` / `"Center map on RELAY"`.
- Clicking a selected row again deselects it (toggle semantics).
- Clicking a relay row centers the tactical map on that relay.

---

## 4. Screenshots

All screenshots are in `./screenshots/`, captured at 1600×950 unless noted.

### Screenshot 1 — Tactical View, default state
![Tactical default](screenshots/01-tactical-default.png)

1. Fleet Sidebar header — app title + collapse button
2. `Vehicles` section header
3. Vehicles pseudo-table — Platform / Range / Quality
4. `Relays` section header and relay row
5. `Link Matrix` header + metric checkboxes (Modulation / SNR / RSSI)
6. Mesh matrix table (7×7 including headers) with color-coded cells
7. Frequency row `MESH-A · 2.412 GHz` and matrix legend
8. Coordinate chip (map center coordinates)
9. Viewport title `CIVIL NETWORK MONITORING SYSTEM`
10. `MESH LINKS` toggle button + notification bell with count badge
11. Satellite node `TELS-1`
12. Platform overlay cards (abbreviated name, range badges, signal bars, camera button)
13. Control Room circular node with label
14. Radio link polylines (solid = active radio, dashed = radio-capable)
15. Map legend (bottom-left) and zoom controls (bottom-right)
16. Control Room panel header
17. `MODEM CONVOY 23` collapsed card — quality bar, rate, temperature/CPU chips
18. `RADIO CR VHF-7` collapsed card — power toggle, quality bar, rate, temperature/voltage
19. `CONTROL ROOM MONITORING` section header and chart area
20. Chart series checkboxes (Upload / Download / Bandwidth)
21. `COMPARE MODEM MONITORING` list
22. Footer — Tactical/Logical switch, `CONTROL ROOM` and `SETTINGS` buttons

### Screenshot 2 — Modem panel expanded
![Modem expanded](screenshots/02-modem-expanded.png)

1. Expand chevron rotated (expanded)
2. Channel rows: `SIM 1`–`SIM 4`, `ONEWEB`, `STARLINK`
3. Power toggle per channel (blue when on, grey and non-interactive for `NO SIM` / `UNPLUGGED`)
4. State chip: `CONNECTED` (green outline), `NOT CONNECTED` (red), `NO SIM` / `UNPLUGGED` (grey)
5. Signal bars + percentage per channel
6. Rate cell `xx.x (RX)`, `—` when off
7. Expand affordance (circular blue chevron) on `ONEWEB` / `STARLINK` revealing SINR / RSSI / RSRP
8. `REBOOT` button, bottom-right of the expanded body

### Screenshot 3 — Platform selected (vehicle context)
![Vehicle selected](screenshots/03-vehicle-selected.png)
Right panel header shows the platform name; modem/radio/graph switch to that platform; a Latency series and a right-hand `ms` axis appear; the compare list is hidden.

### Screenshot 4 — Settings dialog, System tab
![Settings system tab](screenshots/04-settings-system-tab.png)
1. Header: settings icon, `SYSTEM SETTINGS`, close icon button
2. Tab bar: System / Satellite / Radio / Cellular
3. `Precheck` field group with `Run` contained button and helper text

### Screenshot 5 — Settings dialog, Radio tab
![Settings radio tab](screenshots/05-settings-radio-tab.png)
Radio frequency (number, step 0.025, min 30, max 6000), Channel bandwidth (select), Mesh network ID (text), TX power (number).

### Screenshot 6 — Logical View
![Logical topology](screenshots/06-logical-topology.png)
1. `PLATFORMS` column caption
2. `DIRECT · ROUTER GROUP` cluster (blue border)
3. Edge labels (`CELLULAR → ROUTER`)
4. Topology platform cards, 220 px
5. `RELAY GROUP` cluster (orange dashed border)
6. `NETWORK NODES` column — Router and Relay nodes
7. `CONTROL ROOM` column — TELS-1 satellite node
8. Control Room container with internal `J8` module (`ACTIVE`)
9. Orthogonal elbow edges terminating at J8
10. `CONNECTIVITY MAP` floating panel (bottom-right, collapsed)
11. Topology legend (bottom)

### Screenshot 7 — Control Room drawer (Logical view)
![Control room drawer](screenshots/07-control-room-drawer.png)
Right drawer, 360 px, opened by clicking the Control Room node.

### Screenshot 8 — Camera window
![Camera window](screenshots/08-camera-window.png)
Modal window with backdrop; header title + `H/M/L` quality button + close; road feed image; footer metadata.

### Screenshot 9 — Confirm power change dialog
![Confirm power dialog](screenshots/09-confirm-power-dialog.png)
`CONFIRM POWER CHANGE` title, body text naming the component and the target state, `Cancel` / `Confirm` actions.

### Screenshot 10 — Sidebar collapsed
![Sidebar collapsed](screenshots/10-sidebar-collapsed.png)
32 px vertical rail with chevron and rotated title; the map gains the freed width.

### Screenshot 11 — Tablet width (768 px)
![Tablet](screenshots/11-tablet-768.png)

### Screenshot 12 — Mobile width (390 px)
![Mobile](screenshots/12-mobile-390.png)
Both narrow widths show the same three fixed columns; the center viewport is squeezed and the right panel is pushed off-screen. See §9.

---

## 5. Component Specifications

### 5.1 Buttons

There is no custom `Button` component. All buttons are MUI `Button` / `IconButton` / `ButtonBase` restyled per usage through `styled()`. Theme-level defaults (`src/theme/tacticalTheme.ts`):

```
MuiButtonBase: disableRipple = true
MuiButton root:
  borderRadius: 4
  minHeight: 32
  transition: transform 120ms, box-shadow 120ms, opacity 120ms
  hover: transform translateY(-1px)
  .Mui-disabled: opacity .45, cursor not-allowed
MuiButton typography (theme.typography.button):
  fontFamily Exo, fontSize .75rem (12px), textTransform uppercase, letterSpacing .12em
MuiIconButton root: borderRadius 4
```

| Variant | Where | Height | Padding | Font | Border | Colors | Hover |
|---|---|---|---|---|---|---|---|
| **Footer outlined (Settings)** | `SettingsButton` | ≥32 px | `6px 10px` | 12 px / 600 | 1 px `divider` | `text.secondary` | text `text.primary`, border `alpha(primary,.5)` |
| **Footer outlined active (Control Room)** | `ControlRoomButton` | ≥32 px | `6px 10px` | 12 px / 700 | 1 px `primary.main` when active | active: `primary.main` text, `alpha(primary,.14)` fill | border `alpha(primary,.6)` |
| **REBOOT** | `RebootButton` | ≥32 px | `4px 8px` | 12 px / 600, `letterSpacing .16em` | 1 px `divider` | `text.secondary`; while rebooting `primary.main` | `primary.main` text + `alpha(primary,.1)` fill |
| **Mesh Links** | `LinksButton` | ≥32 px | `4px 8px` | 12 px | 1 px `primary.main` (active) / `divider` | active: `primary.main` bg, `contrastText` label | active: `primary.dark`; inactive: `alpha(paper,.95)` |
| **Contained primary (Run, Confirm)** | MUI default | ≥32 px | MUI `size="small"` | 12 px | none | `primary.main` bg, `#121212` text | `primary.dark` |
| **Text / inherit (Cancel, Close)** | MUI default | ≥32 px | MUI `size="small"` | 12 px | none | `text.primary` | `action.hover` |
| **Icon button** | `CollapseIconButton`, `CloseButton`, `WheelToggle` | 22–28 px | `2–4px` | icon 0.8–0.9rem | 1 px `divider` | `text.secondary` | `text.primary`, border `alpha(primary,.6)` |
| **Quality chip button (H/M/L)** | `QualityButton` | 22 px | `0 4px` | 12 px, `letterSpacing .12em` | 1 px `alpha(primary,.7)` | `primary.main` | `UNKNOWN / NEEDS VERIFICATION` |

Icon sizing inside buttons is set per component: 0.75rem (Run), 0.8rem (Reboot), 0.85rem (Mesh Links), 0.9rem (Settings / Control Room / collapse). **Gap** between icon and label is MUI's `startIcon` default (8 px).

No `Danger` button variant exists. Destructive intent is expressed through the confirmation dialog, not through button color.

### 5.2 `PowerToggle` (GLOBAL)

`src/components/GLOBAL/PowerToggle/`

| Property | Value |
|---|---|
| Track | 26 × 14 px, `borderRadius: 999` |
| Track ON | border `alpha(primary,.7)`, background `alpha(primary,.3)`, `boxShadow: 0 0 6px alpha(primary,.5)` |
| Track OFF | border `divider`, background `action.disabledBackground` |
| Thumb | 10 × 10 px circle, `top:1`, `left: 1 → 13` |
| Thumb ON | `primary.main`; OFF: `text.secondary` |
| Transition | `theme.transitions.create([...])` |
| Semantics | `role="switch"`, `aria-checked`, `aria-label` required |

Props: `checked`, `onChange(checked)`, `label` (required, accessible name), `disabled?`, `confirm?` (default **true**), `lastActive?` (default false).

Behaviour: click always opens the confirmation dialog when `confirm` is true. When `checked && lastActive`, the dialog shows title `ACTION BLOCKED` with an outlined warning Alert reading exactly:

> Cannot turn off this communication link. It is the last remaining active communication range.

and offers only `Close`. Otherwise the title is `CONFIRM POWER CHANGE` with `Cancel` / `Confirm`.

### 5.3 `SignalBars` (GLOBAL) — the standard quality indicator

| Property | Value |
|---|---|
| Notches | 5 (prop `bars`) |
| Notch size | width 3 px, `borderRadius 1`, height `4 + level*2` px (4/6/8/10/12) |
| Track | `height: 12px`, `gap: 1.5px`, `alignItems: flex-end` |
| Filled color | `palette.status[qualityStatus(value)]` + `boxShadow 0 0 4px alpha(tone,.5)` |
| Empty color | `alpha(text.secondary, .22)` |
| Value label | 12 px, `fontWeight 700`, `tabular-nums`, colored with the tone, format `NN%` |
| Disabled | tone = `text.secondary`, zero notches filled |
| Semantics | `role="progressbar"` with `aria-valuenow/min/max` and `aria-label` |

`qualityStatus` / `rateStatus` thresholds live in `src/lib/linkStatus.ts` — always use them, never re-derive thresholds inline.

### 5.4 `QualityMeter` (GLOBAL) — filled bar with inline percentage

Used in the Fleet Sidebar lists and as the headline meter in `ModemPanel` / `RadioPanel`. Renders a full-width track with a status-colored fill and the percentage typeset **inside** the fill. Use it for a headline/row-level score; use `SignalBars` for dense per-channel rows.

### 5.5 `HealthMetrics` (GLOBAL)

Icon + number chips for temperature (°C), CPU (%) and voltage (V). `dense` prop for panel headers. Warning/critical state colors the **text and border only** — never a filled background.

### 5.6 `RebootButton` (GLOBAL)

- Default duration **10 s** (`seconds` prop).
- Idle label `REBOOT` with `PowerSettingsNewIcon`; during the cycle the label becomes `REBOOTING Ns` and the control is `disabled` with `aria-busy`.
- A `::before` progress fill sweeps across the control as the countdown advances.
- Emits `onStart`, `onRebootingChange(boolean)` and `onComplete`. Consumers must disable dependent power toggles while `rebooting` is true (`ModemPanel` does exactly this).
- `minWidth: 104px`, `marginLeft: auto`.

### 5.7 `CollapseButton` (GLOBAL)

Chevron expand/collapse control used by `ModemPanel` and `RadioPanel` headers. Props: `expanded`, `onToggle`, `label`. Exposes an accessible name of the form `"<asset name> channels"`.

### 5.8 `SeriesCheckbox` (GLOBAL)

Checkbox + colored dash + label. Props: `label`, `color`, `checked`, `onChange(checked)`. Used for chart series, the compare list and matrix metric modes. **Any new checkbox in this system must use this component.**

### 5.9 `SectionHeader` (GLOBAL)

Centered uppercase band that separates every panel section. Props: `title`, `action?`, `emphasis?`. Specs in §3.3. Never write a bespoke section title.

### 5.10 `SidePanel` (GLOBAL)

Props: `side ("left" | "right")`, `width?` (default 300), `header?`, `footer?`, `children`. Any new fixed column must use this.

### 5.11 `ModemPanel`

Root: 1 px `divider` border, radius 4, background `alpha(background.paper, .7)`.

- `HeaderRow`: `display:grid; gridTemplateColumns: 20px 20px minmax(0,1fr); gap: 8px; padding: 8px 10px 4px` → collapse button, asset icon (`SignalCellularAlt`, 1rem, primary), name (0.8125rem / 13 px, `fontWeight 700`, `letterSpacing .1em`, ellipsis).
- `MeterRow`: `padding: 0 10px 8px`, `paddingLeft: 32px`, `gap: 8px` → `QualityMeter` + rate text `NN.N Mbps (RX)` (13 px, tone-colored via `rateStatus`).
- `HealthRow`: dense `HealthMetrics`.
- Expanded body — `ChannelList`: top border, `padding: 6px 10px 8px`, `gap: 4px`.
- `ChannelRow`: `display:grid; gridTemplateColumns: auto 76px 74px 1fr 54px; gap: 6px; fontSize: 0.8125rem` → toggle, name, state chip, signal bars, rate.
- `MetricsRow` (RF details): 3-column grid, `marginLeft: 24px`, `padding: 4px`, 1 px border, background `alpha(background.default, .5)`; cells `SINR dB`, `RSSI dBm`, `RSRP dBW/m²`.
- Channel state labels: `connected → CONNECTED`, `disconnected → NOT CONNECTED`, `absent → NO SIM`, `unplugged → UNPLUGGED`. Only `connected` / `disconnected` channels are powerable; `absent` / `unplugged` render a grey (not red) 0 % and a disabled toggle.
- Last-active protection: when only one powerable channel remains on, its toggle receives `lastActive` and the block dialog appears.

### 5.12 `RadioPanel`

Same visual frame and row rhythm as `ModemPanel`. Differences: power toggle sits in the header **left of the name**; icon is a radio/cell-tower icon; health chips are temperature and voltage only (**no CPU**).

### 5.13 `MonitoringGraph`

| Property | Value |
|---|---|
| Library | recharts `LineChart` in `ResponsiveContainer` (100 % × container height) |
| Margin | `{ top: 6, right: 8 (4 with latency), bottom: 14, left: -8 }` |
| Grid | `stroke: divider`, `strokeDasharray "2 3"` |
| X axis | `dataKey "t"`, numeric, domain `[0, 60]`, ticks `0,10,20,30,40,50,60`, label **Seconds** (`insideBottom`, offset −10) |
| Y axis (`mbps`) | domain `[0, 25]`, ticks `0,5,10,15,20,25`, label **Mbps** (`insideLeft`, angle −90, offset 16) |
| Y axis (`ms`) | only with `withLatency`; right side, domain `[0, 200]`, width 34, label **ms** |
| Series | Upload `status.good #66BB6A`; Download `status.marginal #FFA726`; Latency `secondary.main #C28FF4` (vehicle only); Bandwidth `primary.main #90CAF9`, dashed `5 4` |
| Line | `type "monotone"`, `strokeWidth 1.6`, `dot false`, `isAnimationActive false` |
| Tooltip | `background.paper`, 1 px `divider`, `fontSize 12` |
| Legend | `SeriesCheckbox` per series; toggling hides/shows the line |

### 5.14 `MeshMatrix`

See §7.1.

### 5.15 `PlatformCard`

Two variants driven by one prop, both from the same file:

| | `overlay` (tactical map) | `topology` (logical) |
|---|---|---|
| Width | 104 px (92 px when `dense`) | 220 px |
| Base font | 0.625rem (10 px) | 0.6875rem (11 px) |
| Content | Abbreviated name, kind badges, signal bars + %, camera button, alert badge | Full name, kind badges, quality, rate, latency, modem groups, satellite lock rows |
| Expandable | No | No (always expanded) |
| Selection | 1 px border switches to `primary.main` | Same |

### 5.16 `NotificationTray`

Bell icon button with a count badge (red). Opens a list of notifications from `src/data/notifications.ts`. Alert-bearing platforms also show a red count badge on their map card.

---

## 6. Form Fields

All inputs are MUI `TextField`. Theme defaults:

```
MuiTextField defaultProps: variant "outlined", size "small"
MuiOutlinedInput root: borderRadius 4, background rgba(255,255,255,0.09)
MuiOutlinedInput notchedOutline: borderColor rgba(255,255,255,0.23)
```

Field group pattern (`SettingsDialog.styles.ts`): `FieldLabel` (above the control) → control → optional `FieldHint` (helper text below).

### 6.1 Settings — Satellite tab

| Field | Label | Type | Default | Options / constraints | Required | Validation | Helper text |
|---|---|---|---|---|---|---|---|
| Satellite | `Satellite` | Select | `TELS-1` | `TELS-1`, `TELS-2` | Optional (uncontrolled) | None | "Active spacecraft used for the SATCOM uplink." |
| Beam / Transponder | `Beam / Transponder` | Text | `KA-04` | — | Optional | None | "Transponder assignment for the ground terminal." |
| Symbol rate | `Symbol rate (Msym/s)` | Number | `12` | — | Optional | None | — |
| Modulation | `Modulation` | Select | `8PSK` | `QPSK`, `8PSK`, `16APSK` | Optional | None | — |

### 6.2 Settings — Radio tab

| Field | Label | Type | Default | Constraints | Validation | Helper text |
|---|---|---|---|---|---|---|
| Radio frequency | `Radio frequency (MHz)` | Number, **controlled** | `2412` (from `radioFrequency` prop) | `step 0.025`, `min 30`, `max 6000` | On apply: `Number.parseFloat`; `NaN` is ignored (value unchanged, dialog still closes) | "Mesh operating frequency shared by all platforms." |
| Channel bandwidth | `Channel bandwidth` | Select | `20` | `5 MHz`, `10 MHz`, `20 MHz` | None | — |
| Mesh network ID | `Mesh network ID` | Text | `MESH-A` | — | None | — |
| TX power | `TX power (dBm)` | Number | `27` | — | None | — |

### 6.3 Settings — Cellular tab

Preferred network select (`AUTO` / … / `5G`) and further carrier fields — see `SettingsDialog.tsx`.

### 6.4 Settings — System tab

No inputs. One field group: label `Precheck`, a contained `Run` button with `PlayArrowIcon`, helper text "Runs the full communication precheck sequence."

### 6.5 Field states — current reality

| State | Implemented? | Appearance |
|---|---|---|
| Default | Yes | Outlined, `rgba(255,255,255,.09)` fill, 4 px radius |
| Focus | Yes (MUI default) | Outline switches to `primary.main` |
| Hover | Yes (MUI default) | Outline lightens |
| Disabled | Not used | MUI default would apply |
| Read-only | Not used | — |
| Error | **Not implemented** | No field currently sets `error` / `helperText` error styling |
| Loading | **Not implemented** | — |
| Mobile | Not adapted | Dialog is `maxWidth`-constrained; fields are full-width in their group |

**Rule for new forms:** use `error` + `helperText` on the MUI `TextField` for validation messages (red per `palette.error.main #F44336`), and keep the `FieldLabel` / control / `FieldHint` stacking pattern.

---

## 7. Tables

### 7.1 Link Matrix (`MeshMatrix`) — the only real `<table>`

**Location:** Fleet sidebar, bottom. **Purpose:** pairwise link margin between every node.

Structure: N+1 × N+1 grid where N = 6 (`CR`, `P1`–`P5`, `RLY`).

| Column | Type | Alignment | Width | Behaviour |
|---|---|---|---|---|
| Row header (first column) | Text abbreviation | Left | auto | `MatrixRowHeadCell`; highlighted (`primary.main`) when its node is selected |
| `CR` … `RLY` (6 value columns) | Numeric margin | Center | auto (equal share of 100 %) | Background is the status color; text `common.black`; diagonal renders `—` |

Table specs: `width: 100%`, `fontSize: 0.75rem (12px)`, `borderCollapse` via 1 px `divider` on every cell, cell `height: 20px`, `padding: 0`, `fontWeight: 600`. Head cells: `padding: 4px`, `fontWeight: 400`, `letterSpacing: .08em`.

Above the table: three `SeriesCheckbox` metric modes — **Modulation**, **SNR**, **RSSI** — which change the values displayed (not a filter; a display-mode switch).
Below the table: `FrequencyRow` (`MESH-A · 2.412 GHz`, `padding: 4px 6px`, 1 px border, radius 4) and `MatrixLegend` (GOOD / MARGINAL / POOR with 8 × 8 px, radius 2 swatches).

Not implemented: sorting, filtering, search, pagination, row selection, sticky header, row actions, empty state, loading state, error state, responsive collapse.

### 7.2 Vehicles / Relays lists (flex pseudo-tables)

| Column | Type | Alignment | Width | Behaviour |
|---|---|---|---|---|
| Platform / Relay | Text | Left | **96 px** (`NameCell`, `fontWeight 700`, ellipsis) | Row click selects/deselects |
| Range | Text | Left | **74 px** (`KindCell`, `text.secondary`) | Joined with ` + ` for multi-range platforms |
| Quality | `QualityMeter` | Fill | `flex: 1; minWidth: 0` | Status-colored fill with inline `NN%` |

Head row (`ListHeadRow`): 12 px, uppercase, `letterSpacing .08em`, `text.secondary`, `paddingBottom: 4px`.
Additional cell styles available for reuse: `RateCell` (right-aligned, default 44 px, status-colored) and `SatelliteName`.

---

## 8. Modals, Dialogs and Drawers

### 8.1 `SettingsDialog`

| Property | Value |
|---|---|
| Trigger | `SETTINGS` button, right panel footer |
| Type | MUI `Dialog` (`SettingsDialogRoot`) |
| Position | Centered; overlay = MUI default scrim |
| Radius | 8 px (`MuiDialog.paper` override) |
| Shadow | `0 8px 18px rgba(0,0,0,0.35)` |
| Header | Settings icon (small, primary) + `SYSTEM SETTINGS` title + close `IconButton` (`aria-label="Close settings"`) |
| Tabs | System / Satellite / Radio / Cellular; `aria-label="Settings sections"`; controlled by local `tab` state, default `system` |
| Body | `TabBody` with stacked `FieldGroup`s |
| Footer | `DialogFooterRow` with the apply action (Radio tab applies the frequency and closes) |
| ESC | Closes (MUI default `onClose`) |
| Click outside | Closes (MUI default) |
| Validation | Frequency parse only; invalid input silently keeps the previous value |
| Loading / error | None |

Flow: click SETTINGS → dialog opens on the last-used tab within the session (state persists while the panel is mounted) → user edits → apply → `onRadioFrequencyChange(value)` → dialog closes. No toast, no confirmation.

### 8.2 `ControlRoomDrawer`

| Property | Value |
|---|---|
| Trigger | Click / `Enter` / `Space` on the CONTROL ROOM node in Logical view (`role="button"`, `tabIndex=0`, `aria-label="Open control room parameters"`) |
| Type | MUI `Drawer`, anchored right |
| Width | **360 px** |
| Paper | `padding: 8px`, background `panel.header #212121`, `borderLeft: 1px solid divider`, `backgroundImage: none`, `gap: 6px` |
| Header | `DrawerHeader` — 0.8125rem (13 px), `fontWeight 700`, `letterSpacing .2em`, `primary.main`, space-between with a close control |
| Body | `DrawerStack` — column, `gap: 6px`, `overflowY: auto`; modem, radio, monitoring graph and configuration content |
| Props | `open`, `onClose`, `modemName` (e.g. `"J8"`) |
| ESC / click outside | Close (MUI Drawer defaults) |

### 8.3 `CameraWindow`

| Property | Value |
|---|---|
| Trigger | `CAMERA` icon button on a tactical platform card |
| Backdrop | `position: fixed; inset: 0; zIndex: theme.zIndex.modal` |
| Frame | `width: min(90vw, 680px)`, radius 4, 1 px `alpha(primary,.6)` border |
| Header | `padding: 6px 8px`, 12 px, `letterSpacing .16em`, `primary.main`; title, spacer, `H/M/L` quality button (22 px tall), close icon button |
| Body | Feed image, `width:100%; height:auto` (`src/assets/road-feed.jpg`) |
| Footer | `padding: 6px 8px`, 12 px, `letterSpacing .12em`, `text.secondary` |
| Close | Close icon only. `UNKNOWN / NEEDS VERIFICATION`: ESC and backdrop-click behaviour is not implemented explicitly in the component |

### 8.4 Power confirmation dialog

Owned by `PowerToggle`; see §5.2. `maxWidth="xs"`, `fullWidth`. Title `fontSize: 0.8rem`, `letterSpacing: .12em`; body text `fontSize: 0.75rem`.

Flow: toggle click → dialog → `Confirm` → `onChange(!checked)` → local state updates instantly (no API, no toast) → dialog closes. `Cancel` / ESC / backdrop → no change.

### 8.5 `ModemCompareWindow`

| Property | Value |
|---|---|
| Trigger | Ticking a modem in `COMPARE MODEM MONITORING` |
| Type | `position: fixed`, `zIndex: 1300`, **380 px** wide |
| Frame | 1 px `alpha(primary,.5)`, radius 4, background `panel.surface`, `boxShadow: 0 12px 32px rgba(0,0,0,.55)` |
| Header | `padding: 6px 8px`, 12 px / 700, `letterSpacing .16em`, `primary.main`, `cursor: move`, `userSelect: none` — drag handle |
| Close | Header `CloseButton` (2 px padding, 0.85rem icon) — also unticks the source checkbox |
| Stacking | Each new window offsets by 32 px from `{x:120, y:120}` |

### 8.6 `CoordinateDialog`

MUI Dialog with a `FieldRow` (two fields side by side, `gap: 8px`) and a `Hint` paragraph (13 px, `text.secondary`).

---

## 9. Responsive Design

### 9.1 Breakpoints

The theme does **not** override MUI breakpoints, so the MUI v7 defaults are in force:

| Key | Min width |
|---|---|
| `xs` | 0 |
| `sm` | 600 px |
| `md` | 900 px |
| `lg` | 1200 px |
| `xl` | 1536 px |

**Critical finding:** no application component uses `theme.breakpoints`, media queries, `useMediaQuery`, or MUI `Grid` responsive props. Verified by inspection of every `*.styles.ts` file.

### 9.2 Actual behaviour

| Viewport | Behaviour |
|---|---|
| Desktop ≥ ~1440 px | Intended layout. Sidebar 296 px + viewport flex + panel 340 px. |
| Desktop 1024–1440 px | Works; the center viewport absorbs the reduction. |
| Tablet 768 px (Screenshot 11) | Same three fixed columns. 296 + 340 = 636 px of chrome leaves ~130 px of map. Layout is technically intact but not usable. |
| Mobile 390 px (Screenshot 12) | Sidebar alone exceeds the viewport; the map and right panel are pushed off-screen. `body { overflow: hidden }` prevents horizontal scrolling, so the right panel becomes **unreachable**. |

**Status: the system is desktop-only by construction.** This is existing UX debt, recorded in §19.

### 9.3 Rules for new features

- Do not introduce a responsive pattern that only your feature honours — it will look broken next to the fixed columns.
- New content must live inside one of the three existing columns or in an overlay surface.
- If a responsive strategy is required, it must be introduced globally (collapse the sidebar to its 32 px rail below `md`, convert the right panel into a Drawer) as a dedicated task, not as a side effect of a feature.

---

## 10. Design System

All tokens live in **`src/theme/tacticalTheme.ts`**. There are no CSS custom properties and no Tailwind theme tokens in use for components.

### 10.1 Colors

| Token | Value | Usage |
|---|---|---|
| `palette.primary.main` | `#90CAF9` | Titles, active controls, icons, focus accents |
| `palette.primary.light` | `#BBDEFB` | Emphasised section header title |
| `palette.primary.dark` | `#2477E8` | Active button hover |
| `palette.primary.contrastText` | `#121212` | Text on filled primary |
| `palette.secondary.main` | `#C28FF4` | Latency chart series |
| `palette.error.main` | `#F44336` | Errors |
| `palette.warning.main` | `#FFA726` | Relay pathway, warnings |
| `palette.info.main` | `#29B6F6` | Reserved |
| `palette.success.main` | `#66BB6A` | Reserved |
| `palette.status.good` | `#66BB6A` | Quality ≥ good, Upload series, ACTIVE |
| `palette.status.marginal` | `#FFA726` | Marginal quality, Download series, WARNING |
| `palette.status.poor` | `#F44336` | Poor quality, DEGRADED |
| `palette.background.default` | `#303030` | App background |
| `palette.background.paper` | `#424242` | Cards / surfaces (usually at `alpha(...,0.7)`) |
| `palette.panel.surface` | `#121212` | Side panel bodies |
| `palette.panel.header` | `#212121` | Section headers, panel headers/footers, drawer |
| `palette.panel.drawer` | `#292929` | Drawer variant surface |
| `palette.interactive.main` | `#2477E8` | Saturated blue for connected/active states |
| `palette.interactive.glow` | `rgba(47,128,237,0.4)` | Glow around active states |
| `palette.text.primary` | `#FFFFFF` | Primary text |
| `palette.text.secondary` | `rgba(255,255,255,0.7)` | Labels, captions |
| `palette.text.disabled` | `rgba(255,255,255,0.5)` | Disabled text |
| `palette.divider` | `rgba(255,255,255,0.12)` | All borders and separators |
| `palette.action.hover` | `rgba(255,255,255,0.08)` | Hover fill |
| `palette.action.selected` | `rgba(255,255,255,0.08)` | Selected fill |
| `palette.action.disabled` | `rgba(255,255,255,0.3)` | Disabled foreground |
| `palette.action.disabledBackground` | `rgba(255,255,255,0.12)` | Toggle OFF track |

Mode is **`dark` only**. There is no light theme and none should be added.

**Alpha convention:** transparency is always produced with MUI's `alpha()` helper against a token — e.g. `alpha(theme.palette.primary.main, 0.14)`. Common values in use: `.08`, `.1`, `.12`, `.14`, `.18`, `.22`, `.28`, `.3`, `.5`, `.6`, `.7`, `.9`.

### 10.2 Typography

Font family: `"Exo", "Assistant", Arial, sans-serif` (loaded via a `<link>` to Google Fonts in `src/routes/__root.tsx`). Base `typography.fontSize: 12`.

| Role | Size | Weight | Letter spacing | Where |
|---|---|---|---|---|
| Panel title / emphasised section header | 0.875rem — **14 px** | 800 | 0.24em | `SectionHeader emphasis`, sidebar `h1` (0.2em) |
| Standard section header | 0.75rem — **12 px** | 800 | 0.2em | `SectionHeader` |
| Asset name | 0.8125rem — **13 px** | 700 | 0.1em | `ModemPanel`/`RadioPanel` names |
| Body / row text | 0.8125rem — **13 px** | 400–700 | 0.06–0.08em | Channel rows, rate values |
| `body1` | 0.875rem — 14 px | 400 | — | MUI default body |
| `body2` / `caption` | 0.75rem — 12 px | 400 | — | Captions, legends, matrix, chart ticks |
| Small labels | 0.75rem — **12 px** | 400–700 | 0.08–0.16em | List heads, legends, chips, button labels |
| Map card text | 0.625rem — **10 px** | 400–800 | 0.06–0.08em | `PlatformCard variant="overlay"` |
| Topology card text | 0.6875rem — **11 px** | 400–800 | 0.06em | `PlatformCard variant="topology"` |
| Button | 0.75rem — 12 px | 600–700 | 0.12em | uppercase, theme default |
| Global body | — | — | `letterSpacing: 0.01em` | `MuiCssBaseline` |

Numeric readouts use `fontVariantNumeric: "tabular-nums"`.

### 10.3 Spacing

`theme.spacing = 4` — **`spacing(n) = n × 4px`**.

| `spacing()` | px | Typical use |
|---|---|---|
| 0.5 | 2 | Card inner padding (map cards), micro gaps |
| 0.75 | 3 | Badge gaps |
| 1 | 4 | Chip padding, tight gaps |
| 1.5 | 6 | Control gaps, header padding |
| 2 | 8 | Standard gap, list row gap |
| 2.5 | 10 | Panel stack padding/gap |
| 3 | 12 | Panel/section padding, legend offsets |
| 6 | 24 | Metrics row indent |
| 8 | 32 | Meter row left indent |

### 10.4 Border radius

`theme.shape.borderRadius = 4`.

| Element | Radius |
|---|---|
| Buttons, icon buttons, inputs, chips, badges, cards, panels, matrix frame | **4 px** (`theme.shape.borderRadius`) |
| Paper (`MuiPaper.rounded`), Dialogs | **8 px** |
| Toggle track | `999` (pill) |
| Toggle thumb, wheel nodes | `50%` |
| Signal bar notch | 1 px |
| Legend swatch | 2 px |
| Scrollbar thumb | 12 px |

### 10.5 Shadows

| Shadow | Value | Usage |
|---|---|---|
| Dialog | `0 8px 18px rgba(0,0,0,0.35)` | `MuiDialog.paper` |
| Floating window | `0 12px 32px rgba(#000, 0.55)` | `ModemCompareWindow` |
| Toggle ON glow | `0 0 6px alpha(primary, .5)` | `PowerToggle` |
| Signal notch glow | `0 0 4px alpha(tone, .5)` | `SignalBars` filled notch |

MUI's default elevation shadows are otherwise unused; surfaces are separated by 1 px `divider` borders instead.

### 10.6 Icons

Library: **`@mui/icons-material`** exclusively. No SVG assets, no other icon set.

| Size | Where |
|---|---|
| 0.75rem (12 px) | Kind badges, Run button icon |
| 0.8rem (13 px) | Reboot icon, wheel toggle, coordinate chip |
| 0.85rem (14 px) | Mesh Links icon, compare window close |
| 0.9rem (14–15 px) | Settings / Control Room / collapse buttons |
| 1rem (16 px) | Asset icons, map node badges |
| 1.05rem (17 px) | Channel expand chevron (circular, primary-tinted) |

Icon color is `text.secondary` by default and `primary.main` for asset/active icons.

| Action | Icon |
|---|---|
| Cellular modem | `SignalCellularAlt` |
| Radio | `SettingsInputAntenna` / cell-tower icon |
| Satellite | `SatelliteAlt` |
| Router | `Router` |
| Modem module | `Memory` |
| Control room / hub | `Hub` |
| Settings | `Settings` |
| Close | `Close` |
| Expand / collapse | `ExpandMore` / `ExpandLess`, `KeyboardDoubleArrowLeft` / `Right` |
| Reboot | `PowerSettingsNew` |
| Run | `PlayArrow` |
| Mesh links | `Link` |
| Expand window | `OpenInFull` / `CloseFullscreen` |

### 10.7 Scrollbars

```
*::-webkit-scrollbar        { width: 5px; height: 5px }
*::-webkit-scrollbar-thumb  { background: #9E9E9E; border-radius: 12px }
*::-webkit-scrollbar-track  { background: transparent }
```

### 10.8 Motion

`MuiButton` transition: `transform 120ms ease, box-shadow 120ms ease, opacity 120ms ease`; hover lifts by `translateY(-1px)`. Everything else uses `theme.transitions.create([...])` with MUI defaults. Ripple is globally disabled.

---

## 11. States

| State | Convention |
|---|---|
| **Default** | 1 px `divider` border, `alpha(background.paper, .7)` fill, `text.secondary` text |
| **Hover** | Border → `alpha(primary, .5–.6)`, text → `text.primary` or `primary.main`, fill → `alpha(primary, .08–.1)`; buttons lift 1 px |
| **Focus** | MUI default focus-visible ring. **No custom focus styles are defined** — see §14 |
| **Active / selected** | `primary.main` border, `alpha(primary, .12–.18)` fill, `primary.main` text; list rows add a 2 px left border |
| **Disabled** | `opacity .45`, `cursor: not-allowed` (buttons); `action.disabledBackground` (toggle track); grey text for absent hardware |
| **Loading** | Only the REBOOT countdown: label `REBOOTING Ns`, control disabled, `aria-busy`, sweeping progress fill |
| **Error** | Status color `#F44336` applied to **text and border only**, never as a fill (health chips, rate values, matrix cells being the exception where cells are filled with black text) |
| **Warning** | `#FFA726`, same outline-only rule |
| **Empty** | Not implemented anywhere |
| **Blocked** | `ACTION BLOCKED` dialog with an outlined warning Alert (last-remaining-link rule) |
| **Dimmed** | Logical view non-active routes: `opacity 0.18` on edges, reduced opacity on cards |
| **Off / inactive** | Rate shows `—`, signal bars show 0 filled notches in `text.secondary` |

---

## 12. UX Flows

### 12.1 Select a platform

1. Operator clicks a row in **Vehicles**, a card on the map, or a card in the topology.
2. `selectedVehicleId` is set in `src/routes/index.tsx`; `selectedRelayId` is cleared.
3. Sidebar row gains the selected treatment (2 px left border + tint).
4. Map/topology card border becomes `primary.main`.
5. Right panel header switches to the platform label (emphasised style).
6. Modem, radio and graph data are recomputed; the Latency series and right `ms` axis appear; the compare list is hidden.
7. Clicking the same row/card again clears the selection and restores the CONTROL ROOM context.

### 12.2 Power a channel off (with redundancy protection)

1. Operator clicks a `PowerToggle`.
2. Confirmation dialog opens: `CONFIRM POWER CHANGE`, body naming the component and target state.
3. `Confirm` → local state flips; the channel shows `NOT CONNECTED`, 0 % bars and `—` rate. `Cancel` / ESC / backdrop → no change.
4. If the toggle was the **last active** powerable channel, step 2 instead shows `ACTION BLOCKED` with the fixed warning text and only a `Close` action; no state change is possible.

### 12.3 Reboot a modem

1. Operator expands the modem panel (`CollapseButton`) — REBOOT is only visible when expanded.
2. Click `REBOOT` → `onRebootingChange(true)`; the button is disabled, label becomes `REBOOTING 10s`, progress fill sweeps.
3. All channel power toggles in that panel are `disabled` for the duration.
4. Countdown decrements every 1000 ms.
5. At 0: `onRebootingChange(false)` then `onComplete(modemId)`; the button returns to `REBOOT`; toggles re-enable.
6. There is no failure path — the cycle is purely client-side and always succeeds.

### 12.4 Change the radio frequency

1. Footer → `SETTINGS` → dialog opens.
2. Select the **Radio** tab → edit `Radio frequency (MHz)`.
3. Apply → `Number.parseFloat`; a valid number is propagated via `onRadioFrequencyChange`; the dialog closes either way.
4. No toast, no persistence, no server call.

### 12.5 Switch Tactical ↔ Logical

1. Footer `ViewModeSwitch` (exclusive toggle group).
2. `mode` changes in the route component.
3. The viewport swaps `TacticalMap` ↔ `LogicalTopology`. `MESH LINKS` disappears in Logical; `CONNECTIVITY MAP` appears.
4. The current selection is preserved across modes.

### 12.6 Open a camera feed

1. Click the `CAMERA` icon on a tactical platform card.
2. A backdrop plus a `min(90vw, 680px)` window opens with the feed and an `H/M/L` quality control.
3. Close via the header `X`.

### 12.7 Inspect Control Room parameters (Logical)

1. Click (or focus + `Enter`/`Space`) the CONTROL ROOM node.
2. A 360 px right Drawer slides in with modem, radio, monitoring and configuration content.
3. ESC or backdrop click closes it.

### 12.8 Error flow

**None exists.** There are no asynchronous operations in the system today. Any new feature that introduces one must define its own error flow (see §17).

---

## 13. Empty / Loading / Error States

| Screen / region | Loading | Empty | Error |
|---|---|---|---|
| Tactical map | None (image loads natively) | N/A — static fixtures | None |
| Logical topology | Edges appear after the first layout measurement (one frame) | N/A | None |
| Vehicles / Relays lists | None | **Not implemented** — must be added if the list can ever be empty | None |
| Link Matrix | None | Not implemented | None |
| Modem / Radio panels | REBOOT countdown only | Channels with no hardware use `NO SIM` / `UNPLUGGED` grey rows — this is the closest existing empty-ish pattern | None |
| Monitoring graph | None; `ResponsiveContainer` renders nothing until it has measured its box | Not implemented | None |
| Settings dialog | None | N/A | None |

**Recommended patterns for new features (no precedent exists — follow these to stay consistent):**
- *Loading:* reuse the REBOOT pattern — disable the control, show an in-place textual state, set `aria-busy`. If a region must show progress, use MUI `LinearProgress` with `primary.main`.
- *Empty:* a centered 12 px `text.secondary` message inside the section body, with the section's `SectionHeader` still rendered. No illustrations — none exist in the system.
- *Error:* MUI `Alert severity="error" variant="outlined"` (the outlined variant is already used for the blocked-toggle warning), plus a `Retry` outlined button styled like `SettingsButton`.

---

## 14. Accessibility

### 14.1 Implemented today

| Concern | Status |
|---|---|
| Switch semantics | `PowerToggle` uses `role="switch"`, `aria-checked`, mandatory `aria-label` |
| Progress semantics | `SignalBars` uses `role="progressbar"` with `aria-valuenow/min/max` and `aria-label` |
| Toggle-button pressed state | `MESH LINKS` sets `aria-pressed` |
| Busy state | `RebootButton` sets `aria-busy` and `disabled` |
| Expanded state | Channel RF buttons set `aria-expanded`; `CollapseButton` carries a descriptive label |
| Labels on icon-only controls | Present: `Collapse panel`, `Expand panel`, `Close settings`, `Open control room parameters`, `Select PLATFORM n`, `Center map on RELAY`, `<channel> RF details` |
| Keyboard on custom clickables | The Logical CONTROL ROOM node handles `Enter` and `Space` with `role="button"` and `tabIndex=0` |
| Dialog labelling | `SettingsDialog` uses `aria-labelledby="settings-title"` |
| Semantic landmarks | `main` (layout), `aside` (side panels), `header` / `footer`, `h1` for the app title |
| Tabs | MUI `Tabs` with `aria-label="Settings sections"` (full ARIA tab semantics from MUI) |

### 14.2 Gaps to close in new work

| Gap | Requirement for new features |
|---|---|
| Focus visibility | `disableRipple` is global and no custom `:focus-visible` styles exist. New interactive elements must define a visible focus style — recommended: `outline: 2px solid palette.primary.main; outline-offset: 2px`. |
| Map card `role="button"` divs | Several clickable `div`s lack keyboard handlers. New clickable elements must be `ButtonBase`/`button`, or carry `role="button"`, `tabIndex=0` **and** `Enter`/`Space` handlers. |
| Contrast | 10 px `text.secondary` (`rgba(255,255,255,.7)`) map card text over a photographic map is likely below WCAG AA. `NEEDS VERIFICATION`. Do not go below 10 px, and prefer ≥ 12 px for any new text. |
| Status by color alone | Link status is conveyed by color in several places. New status indicators must pair color with a text or shape cue (as `SignalBars` does with its `NN%` label). |
| Live regions | None exist. Asynchronous results in new features should be announced via `aria-live="polite"`. |
| Form errors | No field currently wires `aria-describedby`. New forms must use MUI `TextField error` + `helperText`, which wires it automatically. |

---

## 15. Component Architecture

### 15.1 Shared components (`src/components/GLOBAL/`) — always reuse

| Component | Purpose | Props | Variants / states | Used in |
|---|---|---|---|---|
| `SidePanel` | Fixed side column with pinned header/footer and scrollable body | `side`, `width?`(300), `header?`, `footer?`, `children` | left / right | `FleetSidebar`, `ControlRoomPanel` |
| `SectionHeader` | Centered uppercase section band | `title`, `action?`, `emphasis?` | normal / emphasis | Both panels |
| `PowerToggle` | On/off switch with confirmation and last-link protection | `checked`, `onChange`, `label`, `disabled?`, `confirm?`(true), `lastActive?` | on / off / disabled / blocked | `ModemPanel`, `RadioPanel`, `CommsLinkRow` |
| `SignalBars` | Compact 5-notch quality indicator | `value`, `disabled?`, `ariaLabel?`, `showValue?`(true), `bars?`(5) | good / marginal / poor / disabled | Channel rows, `PlatformCard` |
| `QualityBar` | Filled quality bar | `value`, `disabled?`, `ariaLabel?` | good / marginal / poor / disabled | `CommsLinkRow` |
| `QualityMeter` | Filled bar with inline `NN%` | `value`, `ariaLabel?` | good / marginal / poor | Sidebar lists, panel meter rows |
| `HealthMetrics` | Temperature / CPU / voltage chips | `temperature`, `cpu?`, `voltage?`, `dense?` | normal / warning / critical (outline only) | `ModemPanel`, `RadioPanel`, `PlatformCard` |
| `RebootButton` | 10 s reboot with countdown | `seconds?`(10), `label?`, `onStart?`, `onComplete?`, `onRebootingChange?` | idle / rebooting | `ModemPanel` |
| `CollapseButton` | Expand/collapse chevron | `expanded`, `onToggle`, `label` | expanded / collapsed | `ModemPanel`, `RadioPanel` |
| `SeriesCheckbox` | Checkbox + color dash + label | `label`, `color`, `checked`, `onChange` | checked / unchecked | Chart legend, compare list, matrix modes |
| `StatusIndicator` | Status dot/label | see source | good / marginal / poor | Various |

### 15.2 Feature components (`src/components/`)

| Component | Purpose | Key props |
|---|---|---|
| `MonitorLayout` | Three-column flex shell | `children` |
| `FleetSidebar` | Left column | `title`, `open`, `onOpenChange`, `selectedVehicleId?`, `onSelectVehicle?`, `selectedRelayId?`, `onSelectRelay?` |
| `MonitorViewport` | Center column and its chrome | `mode`, `linksOn`, `onLinksOnChange`, `title`, selection props |
| `ControlRoomPanel` | Right column | `mode`, `onModeChange`, `selectedVehicleId`, `onSelectVehicle`, `onRunPrecheck?` |
| `TacticalMap` | Map, markers, links | `linksOn`, selection props |
| `LogicalTopology` | Layered topology graph | `linksOn`, `selectedVehicleId?`, `onSelectVehicle?` |
| `PlatformCard` | Platform data card | `unit`, `variant`, `compact?`, `selected?`, `onSelect?` |
| `ModemPanel` | Modem asset with channels | `modem`, `expanded`, `onExpandedChange`, `onReboot?` |
| `RadioPanel` | Radio asset | `radio`, `expanded`, `onExpandedChange`, `enabled`, `onEnabledChange`, `lastActive` |
| `MonitoringGraph` | Throughput/latency chart | `samples`, `maxBandwidth?`, `withLatency?` |
| `MeshMatrix` | Link margin matrix | — (reads fixtures) |
| `ConnectivityWheel` | Circular connectivity map | — |
| `SettingsDialog` | Tabbed settings | `open`, `onClose`, `radioFrequency`, `onRadioFrequencyChange`, `onRunPrecheck?` |
| `ControlRoomDrawer` | Right drawer | `open`, `onClose`, `modemName` |
| `CameraWindow` | Camera feed window | see source |
| `ModemCompareWindow` | Draggable graph window | `title`, `samples`, `initial`, `onClose` |
| `NotificationTray` | Bell + notification list | — |
| `ViewModeSwitch` | Tactical / Logical toggle | `mode`, `onModeChange` |
| `CommsAssetCard`, `CommsLinkRow`, `ThroughputChart`, `CoordinateDialog`, `RadioPanel` | Supporting | see sources |

### 15.3 Hooks and utilities

| Module | Purpose |
|---|---|
| `src/hooks/useMapViewport.ts` | Zoom/pan state and transform for the tactical map |
| `src/hooks/useMapDrag.ts` | Left-button drag for map nodes and floating windows |
| `src/hooks/useToggleList.ts` | Multi-select list state |
| `src/lib/linkStatus.ts` | `qualityStatus`, `rateStatus`, `parseRate` — **the only place status thresholds are defined** |
| `src/lib/telemetry.ts` | Deterministic latency estimation |
| `src/data/network.ts` | `platforms`, `relays`, `radioLinks`, `meshMargins`, `GROUND_STATION_POSITION`, `SATELLITE_POSITION`, `MAX_BANDWIDTH_MBPS = 22` |
| `src/data/modems.ts` | `controlRoomModem`, `controlRoomRadio`, `vehicleModem(id)`, `vehicleRadio(id)`, `monitoringSamples(seed)` |
| `src/data/notifications.ts` | Notification fixtures |

### 15.4 State ownership

All cross-column state lives in `src/routes/index.tsx`: `linksOn`, `sidebarOpen`, `mode`, `selectedVehicleId`, `selectedRelayId`. Component-local state (expanded panels, dialog open, compare selection, chart series visibility, per-channel power) lives in the owning component. **New cross-column state belongs in the route component and is passed down as props** — do not add a global store for a single feature.

---

## 16. Code Mapping

| Screen / region | File | Style file | Tokens used |
|---|---|---|---|
| Route + global state | `src/routes/index.tsx` | — | — |
| HTML shell, fonts, theme | `src/routes/__root.tsx` | `src/styles.css` | Exo font link, `ThemeProvider` |
| Design tokens | `src/theme/tacticalTheme.ts` | — | all |
| Layout shell | `src/components/MonitorLayout/MonitorLayout.tsx` | `MonitorLayout.styles.ts` | `background.default` |
| Left column | `FleetSidebar/FleetSidebar.tsx` | `FleetSidebar.styles.ts` | `panel.header`, `primary`, `divider`, `status` |
| Center chrome | `MonitorViewport/MonitorViewport.tsx` | `MonitorViewport.styles.ts` | `primary`, `background.paper` |
| Map | `TacticalMap/TacticalMap.tsx`, `mapGeometry.ts` | `TacticalMap.styles.ts` | `status`, `divider`, `primary` |
| Topology | `LogicalTopology/LogicalTopology.tsx` | `LogicalTopology.styles.ts` | `primary`, `warning`, `status`, `divider` |
| Right column | `ControlRoomPanel/ControlRoomPanel.tsx` | `ControlRoomPanel.styles.ts` | `panel.header`, `primary`, `divider` |
| Modem | `ModemPanel/ModemPanel.tsx` | `ModemPanel.styles.ts` | `background.paper`, `divider`, `status`, `primary` |
| Chart | `MonitoringGraph/MonitoringGraph.tsx` | `MonitoringGraph.styles.ts` (`chartAxisStyles(theme)`) | `status`, `primary`, `secondary`, `divider` |
| Matrix | `MeshMatrix/MeshMatrix.tsx` | `MeshMatrix.styles.ts` | `status`, `divider`, `common.black` |
| Settings | `SettingsDialog/SettingsDialog.tsx` | `SettingsDialog.styles.ts` | MUI dialog/tab/input defaults |
| Drawer | `ControlRoomDrawer/ControlRoomDrawer.tsx` | `ControlRoomDrawer.styles.ts` | `panel.header`, `primary`, `divider` |

**Chart axis styling** must go through the shared `chartAxisStyles(theme)` helper in `MonitoringGraph.styles.ts` — do not inline recharts tick/label styles in a new chart.

---

## 17. Rules for New Features

1. **Reuse before creating.** If a `GLOBAL` component covers the need (`PowerToggle`, `SignalBars`, `QualityMeter`, `SectionHeader`, `SidePanel`, `SeriesCheckbox`, `HealthMetrics`, `RebootButton`, `CollapseButton`), use it. Creating a parallel implementation is a defect.
2. **Folder convention is mandatory.** `ComponentName/ComponentName.tsx` + `ComponentName/ComponentName.styles.ts` + `ComponentName/index.ts`. Reusable → `src/components/GLOBAL/`.
3. **No hard-coded design values.** Every color, spacing, radius, and font value comes from `theme.palette` / `theme.spacing` / `theme.shape` / `theme.typography`. No hex literals, no `px` spacing outside the 4 px scale, no Tailwind color classes in components.
4. **No new colors.** Use the existing palette and `status` triad. Transparency via `alpha(token, n)`.
5. **Radius:** 4 px for everything except Paper/Dialog (8 px) and pills (999).
6. **Typography:** 10 px only for map cards, 11 px for topology cards, 12 px for labels/captions, 13 px for rows and asset names, 14 px for section titles. Uppercase + letter-spacing for all titles, headers and buttons.
7. **Status semantics:** green = good, orange = marginal, red = poor — always via `qualityStatus` / `rateStatus` from `src/lib/linkStatus.ts`. Never invent thresholds.
8. **Warning/error visuals are outline-only** (colored text + border), never filled backgrounds — except matrix cells, which are the documented exception.
9. **Every power state change must be confirmed** with `PowerToggle`'s dialog, and the last-remaining-link rule must be honoured with the exact existing message.
10. **Overlay patterns:** center dialog for settings/confirmations; right Drawer (360 px) for parameter inspection; fixed draggable window for floating charts/feeds. Do not invent a fourth.
11. **Dragging is left-button only** and must not conflict with map panning — reuse `useMapDrag`.
12. **Responsive:** do not add a lone responsive behaviour; the system is fixed-width desktop (see §9.3).
13. **Icons:** `@mui/icons-material` only, sized per §10.6.
14. **No toasts** — there is no toast system. Feedback is in-place (state change, dialog, disabled control). Adding a toast system is a platform decision, not a feature decision.
15. **New cross-column state goes into `src/routes/index.tsx`** and flows down as props.
16. **Accessibility floor:** accessible name on every icon-only control, `role`/`aria-*` matching the widget, keyboard operation for anything clickable, visible focus.
17. **Loading / empty / error:** if the feature can be in those states, implement all three using the recommended patterns in §13.

---

## 18. Feature Implementation Template

```markdown
# New Feature Specification

## Feature Name
...

## Route
/ (single-route app) — specify the column or overlay surface:
[ ] Left sidebar section  [ ] Center viewport  [ ] Right panel section
[ ] Dialog  [ ] Right drawer  [ ] Floating window

## Entry Point
Which existing control opens it (button / row / map node)?

## User Flow
1. User action
2. UI response
3. Data/action performed
4. Success state
5. Error state

## Screen Layout
Top → bottom, with widths, paddings (in spacing() units) and alignment.

## Components Used (existing — must reuse)
SidePanel / SectionHeader / PowerToggle / SignalBars / QualityMeter /
HealthMetrics / RebootButton / CollapseButton / SeriesCheckbox / ...

## New Components Required
Name, folder, why no existing component fits.

## Fields
| Field | Label | Type | Default | Required | Validation | Error message | Helper text |

## Buttons
| Button | Variant (footer outlined / contained primary / icon) | Action | Disabled when |

## Validation
Rules and messages.

## Loading State
REBOOT-style in-place disable, or LinearProgress. Specify.

## Empty State
Centered 12px text.secondary message inside the section body.

## Error State
Outlined MUI Alert (severity error) + Retry outlined button.

## Success State
In-place state change (no toast system exists).

## Responsive Behavior
Fixed-width desktop; confirm nothing breaks at 1024px.

## Accessibility
Accessible names, roles, keyboard operation, focus visibility.

## Existing Components To Reuse
Explicit list.

## Design Tokens Used
palette / spacing / shape references only.
```

---

## 19. Design Inconsistencies / Existing UX Debt

| # | Inconsistency | Where | Recommended standard |
|---|---|---|---|
| 1 | **Two quality indicators coexist** — `SignalBars` (notches) and `QualityMeter` / `QualityBar` (filled bar) | Sidebar lists and panel meter rows use bars; channel rows and map cards use notches | Keep both but codify: `QualityMeter` for a headline/row score with a visible percentage, `SignalBars` for dense repeated rows. Do not add a third. |
| 2 | **No responsive behaviour at all** | Whole app | Desktop-only is the current standard. If mobile is ever required, do it globally: sidebar → rail below `md`, right panel → Drawer. |
| 3 | **Dead style exports** — `PanelHeader`, `PanelTitle`, `PrecheckBar`, `PrecheckLabel`, `RunButton`, `SectionBody`, `AssetStack`, `HealthCaption`, `HealthScale` in `ControlRoomPanel.styles.ts` are unused since PRECHECK moved into Settings | `ControlRoomPanel.styles.ts` | Delete in a cleanup task; do not build on them. |
| 4 | **`maxBandwidth` prop is accepted but ignored** by `MonitoringGraph` (kept for API compatibility) | `MonitoringGraph.tsx` | Either remove the prop or reinstate the reference line — do not pass it expecting an effect. |
| 5 | **Chart can render blank on first paint** in the tactical view (see Screenshot 1 vs Screenshot 6): `ResponsiveContainer` measures a zero-height box until layout settles | `MonitoringGraph` inside the right panel | Give the chart container an explicit height in `ChartRoot` / `ChartCanvas` rather than relying on flex measurement. |
| 6 | **Two different collapse affordances** — `CollapseButton` chevron (panels) vs `KeyboardDoubleArrow` icon button (sidebar) vs `OpenInFull` (connectivity wheel) | Multiple | Acceptable as three distinct semantics (expand section / collapse column / enlarge window). Document, don't unify. |
| 7 | **Mixed clickable semantics** — some cards are `div role="button"` without keyboard handlers, others are `ButtonBase` | `PlatformCard`, list rows | Standardise on `ButtonBase` for new clickable surfaces. |
| 8 | **Matrix cells are the only filled status surfaces**, against the outline-only rule established for health chips | `MeshMatrix` | Keep as the documented exception (dense heatmap); do not extend filled status colors elsewhere. |
| 9 | **Panel widths differ** — 296 px left, 340 px right, 360 px drawer, `SidePanel` default 300 px | Layout | Intentional (right column holds denser data). Always pass the width explicitly. |
| 10 | **No focus-visible styling** with ripple globally disabled | All interactive elements | Add a shared focus ring in the theme (`MuiButtonBase` `:focus-visible`) as a cross-cutting task. |

---

## 20. Developer Handoff — Claude Code

### 20.1 The design language in one paragraph

A dark, dense, military/tactical operations console. Pure dark surfaces (`#121212` panels on a `#303030` app background), 1 px `rgba(255,255,255,0.12)` hairline borders instead of shadows, a single light-blue accent (`#90CAF9`) for titles and active controls, and a green/orange/red status triad for all link health. Type is Exo, small (10–14 px), uppercase, and widely letter-spaced for every title, header and button. Geometry is tight: 4 px radius, 4 px spacing scale, 2–12 px paddings. Information density is the priority — nothing is decorative, and every readout is numeric and tabular.

### 20.2 Components that already exist — reuse them

`SidePanel`, `SectionHeader`, `PowerToggle`, `SignalBars`, `QualityBar`, `QualityMeter`, `HealthMetrics`, `RebootButton`, `CollapseButton`, `SeriesCheckbox`, `StatusIndicator`, plus feature components `ModemPanel`, `RadioPanel`, `MonitoringGraph`, `PlatformCard`, `MeshMatrix`, `SettingsDialog`, `ControlRoomDrawer`, `CameraWindow`, `ModemCompareWindow`, `NotificationTray`, `ViewModeSwitch`.

### 20.3 Design tokens

Everything is in `src/theme/tacticalTheme.ts`: `palette.primary/secondary/error/warning/info/success`, custom `palette.status.{good,marginal,poor}`, `palette.panel.{surface,header,drawer}`, `palette.interactive.{main,glow}`, `spacing = 4`, `shape.borderRadius = 4`, Exo typography scale. No CSS variables, no Tailwind tokens in components.

### 20.4 Patterns you must not break

- Three fixed columns; new content lives inside a column or an overlay.
- Confirmation dialog on every power change, including the exact last-link block message.
- Status color always derived from `src/lib/linkStatus.ts`.
- Warning/error as outlined text, not filled blocks.
- Uppercase, letter-spaced, centered `SectionHeader` for every section.
- One folder per component with a separate `.styles.ts`.
- No toasts, no new color, no new radius, no inline hex.

### 20.5 How to build a new feature

1. Decide the surface: sidebar section, viewport overlay, right-panel section, dialog, drawer, or floating window.
2. Compose it from `GLOBAL` components; add a new component only when nothing fits, and put it in its own folder with a `.styles.ts`.
3. Pull every value from the theme.
4. Wire cross-column state through `src/routes/index.tsx`.
5. Implement default, hover, selected, disabled states; add loading/empty/error if the feature can reach them.
6. Give every control an accessible name and keyboard operation.
7. Verify at 1600×950 and 1280×800.

### 20.6 What to check on desktop

Three columns intact at 1280 px and 1600 px; right panel footer stays pinned; panel body scrolls without clipping the footer; the chart has a measurable height; map cards do not overlap illegibly; the topology's measured edges still terminate at the J8 module.

### 20.7 What to check on mobile

Nothing is supported today. If a feature must be mobile-viable, raise it as a separate global responsive task (§9.3) rather than implementing a one-off.

### 20.8 States that must always be implemented

Default, hover, focus (visible), selected, disabled. Plus loading, empty and error whenever the feature can reach them.

### 20.9 New components: allowed vs forbidden

**Allowed:** a genuinely new domain widget (e.g. a spectrum waterfall) in its own folder; a new section inside an existing panel; a new tab in `SettingsDialog`.
**Forbidden:** a new button component, a new switch, a new quality indicator, a new section header, a new dialog shell, a new checkbox, a new side panel, a new chart axis styling.

### 20.10 Checklists

**Before Implementation**
- [ ] Reviewed existing GLOBAL components
- [ ] Reviewed design tokens in `tacticalTheme.ts`
- [ ] Reviewed the relevant screen section in §3
- [ ] Reviewed responsive constraints (§9)

**During Implementation**
- [ ] Reused existing components
- [ ] Reused existing tokens (no hex, no off-scale spacing)
- [ ] Implemented loading state (if applicable)
- [ ] Implemented empty state (if applicable)
- [ ] Implemented error state (if applicable)
- [ ] Respected fixed-width desktop layout
- [ ] Implemented accessibility (name, role, keyboard, focus)
- [ ] Component folder + separate `.styles.ts` + `index.ts`

**Before Delivery**
- [ ] Desktop matches the existing design language
- [ ] All states implemented
- [ ] No unnecessary new components
- [ ] Existing UI patterns preserved (confirmation, status colors, section headers)
- [ ] `bunx tsgo --noEmit` passes
- [ ] No new console errors in the preview
