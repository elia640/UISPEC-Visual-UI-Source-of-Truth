# UISPEC v2 — Visual & UI Source of Truth

Companion to `UI-UX-Technical-Design-Specification-v2.md`.
That document describes **what the system is**. This one is the **implementation contract**: exact tokens, exact measurements, exact rules.
**When the two disagree, UISPEC v2 wins.** When UISPEC v2 and the code disagree, the code is a bug — report it, do not silently diverge.

- Audience: Claude Code / any developer adding features.
- Stack: React 19 + TypeScript + MUI v7 (`@mui/material` + `styled`) + TanStack Start (single route `/`).
- Scope: presentation only. No redesign. Every value below is read from the current codebase.
- Visual references: `/mnt/documents/ui-spec/screenshots-v2/01…14-*.png`.

---

## 0. Non-negotiable rules

1. **Never hard-code a design value.** No hex colors, no raw px for spacing, no font-family strings in components. Use `theme.palette.*`, `theme.spacing()`, `theme.typography.*`, `theme.shape.borderRadius`, `theme.palette.status.*`, `theme.palette.panel.*`, `theme.palette.interactive.*`.
2. **One component = one folder**: `ComponentName.tsx` + `ComponentName.styles.ts` + `index.ts`. All styling lives in `.styles.ts`. No `sx` for structure.
3. **Reusable across screens ⇒ `src/components/GLOBAL/`.** Screen-specific ⇒ `src/components/<Name>/`.
4. **Strictly typed props**, exported as `export interface XProps`. No `any`. Optional booleans read with `?? false` before reaching styled props.
5. **Dark theme only.** No light mode. Ever.
6. **UPPERCASE + letter-spaced** for labels, headers, buttons, status text. Sentence case only in dialog body text and tooltips.
7. **No new colors.** Status semantics are green/orange/red via `theme.palette.status` only.
8. **Minimum readable size 0.75rem (12px).** Nothing smaller ships.
9. **Terminology:** the communication unit is **HALO** (never "modem"/"router" in user-visible text); the central logical node is **HALO SERVER**; the HQ is **CONTROL ROOM**. Legacy identifiers `ModemPanel`, `MDM_*` still exist in code — keep visible strings HALO.

---

## 1. Design tokens (`src/theme/tacticalTheme.ts`)

### 1.1 Palette

| Token | Value | Use |
|---|---|---|
| `primary.main` | `#90CAF9` | Active controls, selection, key values |
| `primary.light` | `#BBDEFB` | Emphasised titles |
| `primary.dark` | `#2477E8` | Pressed / strong accents |
| `secondary.main` | `#C28FF4` | Secondary chart series |
| `error.main` / `warning.main` / `info.main` / `success.main` | `#F44336` / `#FFA726` / `#29B6F6` / `#66BB6A` | MUI semantics |
| `status.good` | `#66BB6A` | Quality ≥ 70, healthy hardware |
| `status.marginal` | `#FFA726` | Quality 40–69 |
| `status.poor` | `#F44336` | Quality < 40, faults |
| `background.default` | `#303030` | Shell / viewport |
| `background.paper` | `#424242` | Cards, dialogs, floating windows |
| `panel.surface` | `#121212` | Side panels |
| `panel.header` | `#212121` | Section header bars |
| `panel.drawer` | `#292929` | Drawers, nested surfaces |
| `interactive.main` | `#2477E8` | Live/connected accent, tactical asset frames |
| `interactive.glow` | `rgba(47,128,237,0.4)` | Glow on live elements |
| `text.primary` / `secondary` / `disabled` | `#FFFFFF` / `rgba(255,255,255,0.7)` / `rgba(255,255,255,0.5)` | |
| `divider` | `rgba(255,255,255,0.12)` | All borders |
| `action.hover` | `rgba(255,255,255,0.08)` | |
| `action.disabledBackground` | `rgba(255,255,255,0.12)` | Off-state toggle track |

Alpha convention (`alpha()` from `@mui/material/styles`): `0.08–0.10` hover/neutral fill · `0.14` badge fill · `0.22` progress fill · `0.30` active track · `0.50–0.70` borders/glow · `0.92` opaque overlay card.

### 1.2 Spacing & shape

- Base unit `spacing: 4` → `theme.spacing(1) = 4px`.
- Rhythm: `0.5` micro · `1` tight · `1.5`/`2` intra-row · `3` panel padding · `4` section gap.
- `shape.borderRadius = 4` for controls, chips, inputs, badges.
- `8px` for Paper, cards, dialogs, floating windows.
- Pills use `borderRadius: 999`.

### 1.3 Typography

Font stack `"Exo", "Assistant", Arial, sans-serif`, base `fontSize: 12`.

| Role | Size | Weight | Tracking | Case |
|---|---|---|---|---|
| Panel primary title (`SectionHeader emphasis`) | `0.875rem` | 800 | `0.24em` | UPPER |
| Section header | `0.75rem` | 800 | `0.2em` | UPPER |
| Body / primary row | `0.875rem` | 400–600 | `0.01em` | as-is |
| Secondary / caption | `0.75rem` | 400–600 | `0.02em` | as-is |
| Numeric metric | `0.75rem` | 700 | `0.02em` | tabular-nums |
| Button | `0.75rem` | 600 | `0.12em` | UPPER |
| Reboot / footer control | `0.75rem` | 600 | `0.16em` | UPPER |
| Chart axis tick | `11–12px` | 400 | — | — |
| Dialog title | `0.8rem` | 500 | `0.12em` | UPPER |
| Topology layer caption | `0.75rem` | 700 | `0.22em` | UPPER |

**Every numeric readout sets `fontVariantNumeric: "tabular-nums"`** so streaming values do not jitter.

### 1.4 Motion

- Use `theme.transitions.create([...])`. Hard-coded durations allowed only where they already exist:
  - buttons `transform/box-shadow/opacity 120ms ease`, hover `translateY(-1px)`;
  - countdown fill `width 1s linear`.
- Ripple is globally disabled (`MuiButtonBase.disableRipple`). Do not re-enable.
- No entrance animations on data rows; only overlays animate (Drawer 260ms, Dialog default).

### 1.5 Global chrome

- Scrollbars 5px, thumb `#9E9E9E`, radius 12, transparent track.
- `body { overflow: hidden }` — the app never scrolls; panels scroll internally.
- Tooltips: `enterDelay 600`, `leaveDelay 150`, bottom placement, arrow, bg `#333`, `0.75rem`.
- Inputs: outlined, `size="small"`, fill `rgba(255,255,255,0.09)`, outline `rgba(255,255,255,0.23)`, radius 4.
- Disabled: `opacity 0.45`, `cursor: not-allowed`.

---

## 2. Layout contract

```text
┌──────────────┬───────────────────────────────┬──────────────────┐
│ FleetSidebar │        MonitorViewport        │ ControlRoomPanel │
│    296 px    │            flex: 1            │      340 px      │
│ (32 px rail) │   tactical  |  logical        │  header/body/    │
│              │   + top-right overlay bar     │  footer          │
└──────────────┴───────────────────────────────┴──────────────────┘
```

- Shell `MonitorLayout`: `display:flex; height:100vh; width:100%; overflow:hidden`, bg `background.default`.
- `SidePanel` (GLOBAL): column flex, `flexShrink: 0`, fixed width, bg `panel.surface`, `1px solid divider` on the inner edge. Optional pinned `header`/`footer`; middle `SidePanelScroll` is `flex:1; minHeight:0; overflowY:auto`.
- Left width **296**, collapsed rail **32**. Right width **340**.
- Panel body padding `theme.spacing(3)`; section header padding `theme.spacing(1.75, 3)`.
- Viewport overlays: top-right bar holds Mesh Links (tactical only) + notification tray; MESH LINKS MAP wheel is logical-only, offset left to clear the right panel; legend is bottom-left (logical) and tactical legend renders only while Mesh Links is ON.

**Rule for new panels:** compose `SidePanel` + `SectionHeader`. Never build a bespoke panel frame.

---

## 3. GLOBAL component contracts

Search `GLOBAL/` before creating anything. A pattern needed twice gets promoted to `GLOBAL/`.

### `SectionHeader`
`{ title: string; action?: ReactNode; emphasis?: boolean }` — centered title, bg `panel.header`, top+bottom `1px divider`; `action` absolutely positioned right at `spacing(3)`.

### `SignalBars`
5 ascending notches, width `3px`, `height = 4 + level*2`, gap `1.5px`, track `12px`, radius 1. Filled bars take the status tone + `0 0 4px alpha(tone,0.5)`; empty bars `alpha(text.secondary,0.22)`. Optional label `0.75rem/700`, tabular-nums, nowrap.

### `QualityBar`
`{ value: 0–100; disabled?; ariaLabel?; showValue? }` — clamped, `role="progressbar"` with `aria-valuenow/min/max`. Fill color `status[qualityStatus(v)]`; disabled renders neutral `text.secondary`, never red. **In the logical view and both side panels `showValue` is ON** — percentages must be visible everywhere quality is shown.

### `QualityMeter`
Bar (`flex:1`, `minWidth:32`) + score badge: `minWidth 38`, padding `spacing(0.25,0.75)`, radius 4, `0.75rem/700`; text/border/fill from `status[tone]` at `1 / 0.5 / 0.14` alpha.

### `HealthMetrics`
`{ temperature?, cpu?, voltage?, dense? }` — outline chips with `Thermostat` / `Memory` / `Bolt` icons, tone from `temperatureStatus` / `cpuStatus` / `voltageStatus`, `title` carrying the full-word meaning. Renders nothing when all values are undefined. `dense` is the compact map/overlay variant.

### `PowerToggle`
Track `26 × 14` radius 999; thumb `10 × 10`, `left: 1 → 13`.
On: border `alpha(primary,0.7)`, fill `alpha(primary,0.3)`, glow `0 0 6px alpha(primary,0.5)`. Off: border `divider`, fill `action.disabledBackground`, thumb `text.secondary`.
Behavior (do not alter): confirmation dialog by default; when it is the **last active link** the dialog becomes a blocking `Alert severity="warning"` with the exact copy
`Cannot turn off this communication link. It is the last remaining active communication range.`
A11y: `role="switch"`, `aria-checked`, `aria-label`.

### `RebootButton`
Outlined, `minWidth 104`, padding `spacing(1,2)`, `0.75rem/600`, tracking `0.16em`, icon `0.8rem`. 10 s countdown; label `REBOOT` → `REBOOTING Ns`; disabled while running but still `primary.main`; `::before` fill sweeps `width: progress%` in `alpha(primary,0.22)` with `transition: width 1s linear`; `aria-busy` while running; reports `onRebootingChange` so the parent **dims all its controls** during the reboot.

### `CollapseButton`
Icon-only, `padding: 0`, icon `0.95rem`, `text.secondary` → `text.primary` on hover, transparent hover bg, `aria-expanded`. **Only render it when a collapsible body exists** (the radio panel has none and therefore no arrow).

### `StatusIndicator`, `SeriesCheckbox`, `SidePanel`
Status dot + label; chart/matrix series checkbox colored by its series color; the panel frame described in §2.

---

## 4. Status semantics (`src/lib/linkStatus.ts`) — single source of truth

| Function | good | marginal | poor |
|---|---|---|---|
| `qualityStatus(0–100)` | ≥ 70 | 40–69 | < 40 |
| `rateStatus(mbps, max = MAX_BANDWIDTH_MBPS)` | < 85 % of max | ≥ 85 % | > max |
| `temperatureStatus(°C)` | < 55 | 55–64 | ≥ 65 |
| `cpuStatus(%)` | < 70 | 70–84 | ≥ 85 |
| `voltageStatus(V)` | 11.9–13.5 | 11.5–11.9 or 13.5–14 | < 11.5 or > 14 |

**Never re-derive a threshold inline.** Import the helper; a new metric gets a new function here.

---

## 5. Entity cards — `PlatformCard`

| Property | `overlay` (tactical map) | `topology` (logical view) |
|---|---|---|
| Width | 104 (92 when `dense`) | 176 |
| Font size | `0.625rem` | `0.75rem` |
| Border | `divider` | `status[status]` |
| Background | `alpha(background.default, .92)` + `blur(4px)` | `alpha(background.paper, .6)` |
| Labels | abbreviated link kinds | **full names, full link-kind words, no ellipsis** |
| Header wrap | single line | may wrap to two lines (`rowGap: spacing(0.5)`) |
| Selected | `0 0 0 2px alpha(primary,.9)` ring | same |

- Kind badges stay neutral grey — only the quality bar carries status color.
- Tactical asset icon + frame use `interactive.main` (the settings blue).
- **Fault badge:** opaque hollow circle — transparent fill, `status.poor` border, `status.poor` number, floated outside the card top-right (`top:-8; right:-6`). No icon, no pulse, no translucency.
- Hover on a card shows the full asset name via tooltip.

---

## 6. Logical view contract (`LogicalTopology`)

Three stacked rows inside `LayerStack` (`width:100%`, `minWidth 720`, `maxWidth 1680`, `height:100%`, `minHeight 460`, `gap spacing(3)`, `padding spacing(2,2,13)`):

1. **CONTROL ROOM** (top) — transparent container so edges can pass through it; contains one or more **HALO** router rectangles (full border, detached from the container's bottom edge, sized to leave room for additional routers).
2. **HALO SERVER** (middle) — every platform's traffic passes through it. It is not a "relay".
3. **PLATFORM row** (bottom) — `flex-nowrap`, `space-evenly`, `columnGap spacing(1.5)`, all units on one line.

Routing rules:
- Every platform connects upward to HALO SERVER; a platform without direct visibility connects to its **peer relay platform first**, then that peer carries the traffic to HALO SERVER.
- HALO SERVER → CONTROL ROOM router draws **one line per incoming vehicle route** — never merged — except that traffic already aggregated through a peer relay travels as a single consolidated trunk.
- Segment colors are independent per segment: `status.good/marginal/poor`.
- Every segment carries a **download-rate label (Mbps)** beside the line.
- Lines are orthogonal/straight and must not cross.
- Selection/hover highlights the related route and dims the rest; clicking empty canvas clears selection and resets to the CONTROL ROOM view.
- Legend sits bottom-left.

---

## 7. State presentation rules

| State | Presentation |
|---|---|
| Inactive / powered off | Value becomes `—`; meters grey (`disabled`); text drops to `text.secondary`; no glow |
| Disabled control | `opacity 0.45`, `cursor: not-allowed` |
| Selected entity | `primary.main` text/border + `alpha(primary,0.1)` fill (cards: 2px ring) |
| Live / connected | `interactive.main` stroke + `interactive.glow` |
| Rebooting | Parent panel content dimmed; REBOOT control stays visible with countdown |
| Hover on topology node | Related route highlighted, unrelated paths dimmed |
| Blocking action | Dialog + `Alert severity="warning" variant="outlined"`, single `Close` |
| Confirmable action | `Cancel` (`color="inherit"`) + `Confirm` (`variant="contained"`) |
| Empty | Short uppercase `text.secondary` line, no illustration |
| Loading | Not present today — new async work uses an inline skeleton or `—`, never a full-screen spinner |

---

## 8. Overlay taxonomy — pick the right one

| Need | Use | Precedent |
|---|---|---|
| Short confirmation / blocking warning | `Dialog`, `maxWidth="xs"`, `fullWidth` | `PowerToggle` |
| Multi-field configuration | Tabbed `Dialog`, `min(760px, 92vw)` | `SettingsDialog` (System / Satellite / Radio / Cellular) |
| Contextual detail for a node | Right drawer, bg `panel.drawer`, 360px, 260ms | `ControlRoomDrawer` |
| Parallel live data | Draggable floating window (Paper, radius 8, header drag handle, close) | `ModemCompareWindow` |
| Media | Portal modal with backdrop, H/M/L quality menu | `CameraWindow` |
| Transient notice | Bell + badge + popover, top-right | `NotificationTray` |

Floating-window contract: drag from the header only (`event.target.closest("button")` guard), clamp to viewport, `setPointerCapture` on the header, position in local state.

---

## 9. Charts (`MonitoringGraph` / `ThroughputChart`, Recharts)

- X axis seconds, Y axis Mbps, optional right axis ms latency.
- Series toggled with `SeriesCheckbox`: Upload, Download, Latency, plus a **dashed** bandwidth-ceiling line.
- Ticks `11–12px` in `text.secondary`; grid uses `divider`.
- Recharts only. Do not add another chart library.

---

## 10. Adding a feature — required sequence

1. **Place it.** Left sidebar = fleet/inventory. Viewport = spatial/topological. Right panel = selected entity's controls & telemetry. Right-panel footer = global mode/settings actions.
2. **Reuse first** — check `GLOBAL/`.
3. **Scaffold** `src/components/<Name>/{<Name>.tsx, <Name>.styles.ts, index.ts}`.
4. **Type and export props**, one-line JSDoc on non-obvious props.
5. **Tokens only.**
6. **Status via `linkStatus.ts`.**
7. **Terminology:** HALO / HALO SERVER / CONTROL ROOM in all user-visible strings.
8. **Accessibility:** `aria-label` on icon-only controls, `role`/`aria-checked` on custom switches, `aria-busy` on async controls, `title` on matrix cells, `aria-expanded` on collapsers.
9. **Verify:** `bunx tsgo --noEmit`, then a preview screenshot at 1600×1000 — nothing overflows, no text under 12px, no ellipsis in the logical view.

### Component template

```ts
// src/components/Example/Example.styles.ts
import { styled } from "@mui/material/styles";

export const ExampleRoot = styled("div")(({ theme }) => ({
  display: "flex",
  alignItems: "center",
  gap: theme.spacing(1.5),
  padding: theme.spacing(1.5, 3),
  backgroundColor: theme.palette.panel.surface,
  borderBottom: `1px solid ${theme.palette.divider}`,
}));

export const ExampleValue = styled("span", {
  shouldForwardProp: (prop) => prop !== "tone",
})<{ tone: string }>(({ tone }) => ({
  fontSize: "0.75rem",
  fontWeight: 700,
  fontVariantNumeric: "tabular-nums",
  color: tone,
}));
```

```tsx
// src/components/Example/Example.tsx
import { useTheme } from "@mui/material/styles";
import { qualityStatus } from "@/lib/linkStatus";
import { ExampleRoot, ExampleValue } from "./Example.styles";

export interface ExampleProps {
  label: string;
  /** Signal quality, 0-100. */
  quality: number;
}

export function Example({ label, quality }: ExampleProps) {
  const theme = useTheme();
  return (
    <ExampleRoot>
      <span>{label}</span>
      <ExampleValue tone={theme.palette.status[qualityStatus(quality)]}>
        {quality}%
      </ExampleValue>
    </ExampleRoot>
  );
}
```

```ts
// src/components/Example/index.ts
export { Example } from "./Example";
export type { ExampleProps } from "./Example";
```

---

## 11. Forbidden patterns

- `sx` for structural layout; inline hex colors; raw px for spacing.
- New fonts, new palettes, light mode, gradients, drop shadows beyond the two defined.
- Text below 12px; truncation in the logical view.
- A second router library, chart library, or styling system (no Tailwind, no CSS modules).
- Full-screen spinners, skeleton pages, toast spam.
- The words "modem" or "router" in user-visible copy.
- Re-deriving status thresholds outside `linkStatus.ts`.
- Rebuilding a panel/dialog/toggle that already exists in `GLOBAL/`.

---

## 12. Visual references

`screenshots-v2/`: `01-tactical-default` · `02-tactical-mesh-off` · `03-halo-panel-expanded` · `04-confirm-power-dialog` · `05-platform-selected` · `06-notifications-open` · `07-settings-system` · `08-settings-radio` · `09-camera-window` · `10-sidebar-collapsed` · `11-logical-default` · `12-logical-platform-selected` · `13-tablet-1024` · `14-mobile-390`.

Known debt visible in the references (do **not** "fix" as a side effect of a feature): mobile/tablet horizontal overflow from the fixed three-column shell; legacy `MODEM/ROUTER` identifiers in code; route metadata title still reads "Comms Network Monitor"; `CardTitle` still declares `nowrap`/ellipsis although topology parents allow wrapping.

---

## 13. Handoff checklist (per change)

- [ ] Folder structure `.tsx` + `.styles.ts` + `index.ts`
- [ ] Props interface exported, no `any`
- [ ] All colors/spacings/sizes from theme tokens
- [ ] Status via `linkStatus.ts`
- [ ] HALO terminology in visible strings
- [ ] Uppercase, letter-spaced labels; nothing under 12px
- [ ] Off/disabled/selected/hover/rebooting states handled
- [ ] `aria-label` / `role` / `aria-busy` where applicable
- [ ] `bunx tsgo --noEmit` clean
- [ ] Screenshot verified at 1600×1000, no overflow or clipping
