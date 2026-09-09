# UISPEC — Visual & UI Source of Truth

Companion to `UI-UX-Technical-Design-Specification.md`.
That document explains **what the system is**. This one is the **implementation contract**: exact tokens, exact measurements, exact rules. When the two disagree, **UISPEC wins**.

Audience: Claude Code / any developer adding features to this application.
Stack: React 19 + TypeScript + MUI v7 (`@mui/material` + `styled`) + TanStack Start.
Scope: presentation only. No redesign. Every value below is read from the codebase.

---

## 0. Non-negotiable rules

1. **Never hard-code a design value.** No hex colors, no raw px for spacing, no font-family strings inside components. Everything comes from the theme: `theme.palette.*`, `theme.spacing()`, `theme.typography.*`, `theme.shape.borderRadius`, `theme.palette.status.*`, `theme.palette.panel.*`, `theme.palette.interactive.*`.
2. **One component = one folder** containing `ComponentName.tsx`, `ComponentName.styles.ts`, `index.ts`. Styles live only in the `.styles.ts`. No `sx` for anything structural (small one-off typography tweaks inside dialogs are the only tolerated exception, and they already exist — do not add more).
3. **Reusable across screens ⇒ `src/components/GLOBAL/`.** Screen-specific ⇒ `src/components/<Name>/`.
4. **Props are strictly typed and exported**: `export interface XProps`. No `any`, no implicit optional booleans — use `flag?: boolean` and read it with `?? false` when passing to styled props.
5. **Dark theme only.** There is no light mode. Do not add one.
6. **Uppercase + letter-spaced** for all labels, headers, buttons, status text. Sentence case only inside dialog body text and tooltips.
7. **No new colors.** Status semantics are green/orange/red only, via `theme.palette.status`.
8. **Minimum readable size is 0.75rem (12px).** Nothing smaller ships.

---

## 1. Design tokens (`src/theme/tacticalTheme.ts`)

### 1.1 Palette

| Token | Value | Use |
|---|---|---|
| `primary.main` | `#90CAF9` | Active controls, selected state, links, key values |
| `primary.light` | `#BBDEFB` | Emphasised section titles |
| `primary.dark` | `#2477E8` | Pressed / strong accents |
| `secondary.main` | `#C28FF4` | Secondary series in charts |
| `success.main` | `#66BB6A` | — |
| `warning.main` | `#FFA726` | — |
| `error.main` | `#F44336` | — |
| `info.main` | `#29B6F6` | — |
| `status.good` | `#66BB6A` | Quality ≥ 70, healthy hardware |
| `status.marginal` | `#FFA726` | Quality 40–69, warning band |
| `status.poor` | `#F44336` | Quality < 40, fault band |
| `background.default` | `#303030` | App shell / viewport |
| `background.paper` | `#424242` | Cards, dialogs, floating windows |
| `panel.surface` | `#121212` | Left & right side panels |
| `panel.header` | `#212121` | Section header bars |
| `panel.drawer` | `#292929` | Drawers, nested surfaces |
| `interactive.main` | `#2477E8` | Connected / live link accent |
| `interactive.glow` | `rgba(47,128,237,0.4)` | Glow around live elements |
| `text.primary` | `#FFFFFF` | |
| `text.secondary` | `rgba(255,255,255,0.7)` | Labels, units, inactive |
| `text.disabled` | `rgba(255,255,255,0.5)` | Disabled |
| `divider` | `rgba(255,255,255,0.12)` | All borders |
| `action.hover` | `rgba(255,255,255,0.08)` | |
| `action.disabledBackground` | `rgba(255,255,255,0.12)` | Off-state toggle track |

Alpha usage convention (`alpha()` from `@mui/material/styles`):
`0.10` hover fill · `0.14` badge fill · `0.22` progress fill · `0.30` active track · `0.50–0.70` borders/glow.

### 1.2 Spacing & shape

- Base unit: `spacing: 4` → `theme.spacing(1) = 4px`.
- Common rhythm: `0.5` (2px) micro, `1` (4px) tight, `1.5`/`2` (6/8px) intra-row, `3` (12px) panel padding, `4` (16px) section gap.
- `shape.borderRadius = 4` for controls, chips, inputs, small elements.
- `8px` for Paper, cards, dialogs, floating windows (`MuiPaper.rounded`, `MuiDialog.paper`).
- Pill controls use `borderRadius: 999`.

### 1.3 Typography

Font stack: `"Exo", "Assistant", Arial, sans-serif`. Base `fontSize: 12`.

| Role | Size | Weight | Letter-spacing | Case |
|---|---|---|---|---|
| Panel primary title (`SectionHeader emphasis`) | `0.875rem` | 800 | `0.24em` | UPPER |
| Section header | `0.75rem` | 800 | `0.2em` | UPPER |
| Body / primary row text | `0.875rem` | 400–600 | `0.01em` | as-is |
| Secondary row / caption | `0.75rem` | 400–600 | `0.02em` | as-is |
| Numeric value / metric | `0.75rem` | 700 | `0.02em` | tabular-nums |
| Button | `0.75rem` | 600 | `0.12em` | UPPER |
| Reboot / footer control | `0.75rem` | 600 | `0.16em` | UPPER |
| Chart axis tick | `11px`–`12px` | 400 | — | — |
| Dialog title | `0.8rem` | 500 | `0.12em` | UPPER |

**All numeric readouts must set `fontVariantNumeric: "tabular-nums"`** so values do not jitter while streaming.

### 1.4 Motion

- Standard: `theme.transitions.create([...])` — never a hard-coded duration except:
- Buttons: `transform 120ms ease, box-shadow 120ms ease, opacity 120ms ease`, hover `translateY(-1px)`.
- Countdown progress fill: `width 1s linear`.
- Ripple is globally disabled (`MuiButtonBase.disableRipple`). Do not re-enable.

### 1.5 Global chrome

- Scrollbars: 5px, thumb `#9E9E9E`, radius 12, transparent track.
- `body { overflow: hidden }` — the app never scrolls; panels scroll internally.
- Tooltips: `enterDelay 600`, `leaveDelay 150`, bottom placement, arrow, bg `#333`, `0.75rem`.
- Inputs: outlined, `size="small"`, fill `rgba(255,255,255,0.09)`, outline `rgba(255,255,255,0.23)`, radius 4.
- Disabled: `opacity 0.45`, `cursor: not-allowed`.

---

## 2. Layout contract

```
┌──────────────┬───────────────────────────────┬──────────────────┐
│ FleetSidebar │        MonitorViewport        │ ControlRoomPanel │
│    296 px    │            flex: 1            │      340 px      │
│ (32 px rail) │  tactical | logical           │                  │
└──────────────┴───────────────────────────────┴──────────────────┘
```

- Shell: `MonitorLayout` → `display: flex; height: 100vh; width: 100%; overflow: hidden`, bg `background.default`.
- `SidePanel` (`GLOBAL`): column flex, `flexShrink: 0`, fixed `width`, bg `panel.surface`, `1px solid divider` on the inner edge (`borderRight` for left, `borderLeft` for right). Optional pinned `header` and `footer`; the middle `SidePanelScroll` is `flex: 1; minHeight: 0; overflowY: auto`.
- Left panel width **296**, collapsed rail **32**. Right panel width **340**.
- Panel body padding: `theme.spacing(3)` (12px). Section header padding: `theme.spacing(1.75, 3)`.

**Rule for new panels:** always compose `SidePanel` + `SectionHeader`. Never build a bespoke panel frame.

---

## 3. Component contracts (GLOBAL)

Use these before writing anything new. If a new pattern is needed 2+ times, promote it to `GLOBAL/`.

### `SectionHeader`
`{ title: string; action?: ReactNode; emphasis?: boolean }`
Centered title, bg `panel.header`, top+bottom `1px divider`. `action` is absolutely positioned right at `spacing(3)`.

### `SignalBars`
5 ascending notches: width `3px`, `height = 4 + level*2` px, gap `1.5px`, track height `12px`, radius `1`. Filled bars take the status tone plus `0 0 4px alpha(tone,0.5)`; empty bars are `alpha(text.secondary, 0.22)`. Optional value label `0.75rem/700`, tabular-nums, tone-colored, `whiteSpace: nowrap`.

### `QualityMeter`
Bar (`flex: 1`, `minWidth: 32`) + score badge: `minWidth 38`, padding `spacing(0.25, 0.75)`, radius 4, `0.75rem/700`, text/border/fill derived from `status[tone]` at `1 / 0.5 / 0.14` alpha.

### `PowerToggle`
Track `26 × 14`, radius 999; thumb `10 × 10` at `left: 1 → 13`.
On: border `alpha(primary,0.7)`, fill `alpha(primary,0.3)`, glow `0 0 6px alpha(primary,0.5)`.
Off: border `divider`, fill `action.disabledBackground`, thumb `text.secondary`.
Behavior (do not alter): confirmation dialog by default; when the toggle is the last active link the dialog becomes a blocking `Alert severity="warning"` with the exact copy
`Cannot turn off this communication link. It is the last remaining active communication range.`
Accessibility: `role="switch"`, `aria-checked`, `aria-label`.

### `RebootButton`
Outlined, `minWidth 104`, padding `spacing(1,2)`, `0.75rem/600`, tracking `0.16em`, icon `0.8rem`. Default 10 s countdown; label `REBOOT` → `REBOOTING Ns`; disabled while running but rendered in `primary.main`; a `::before` fill sweeps `width: progress%` in `alpha(primary,0.22)` with `transition: width 1s linear`. Reports `onRebootingChange` so the parent locks its controls. `aria-busy` while running.

### `CollapseButton`
Icon-only, `padding: 0`, icon `0.95rem`, `text.secondary` → `text.primary` on hover, transparent hover background.

### `StatusIndicator`, `HealthMetrics`, `SeriesCheckbox`, `QualityBar`
Status dot + label; temperature/voltage pairs (`dense` variant for panels); chart/matrix series checkboxes colored by their series color; legacy bar meter.

---

## 4. Status semantics (`src/lib/linkStatus.ts`) — single source of truth

| Function | good | marginal | poor |
|---|---|---|---|
| `qualityStatus(0–100)` | ≥ 70 | 40–69 | < 40 |
| `rateStatus(mbps, max)` | < 85 % of max | ≥ 85 % | > max |
| `temperatureStatus(°C)` | < 55 | 55–64 | ≥ 65 |
| `cpuStatus(%)` | < 70 | 70–84 | ≥ 85 |
| `voltageStatus(V)` | 11.9–13.5 | 11.5–11.9 or 13.5–14 | < 11.5 or > 14 |

**Never re-derive a threshold inline.** Import the helper; if a new metric needs a scale, add a function here.

---

## 5. State presentation rules

| State | Presentation |
|---|---|
| Inactive / powered off | Value replaced by `—`; meters greyed (`disabled`); row text drops to `text.secondary`; no glow |
| Disabled control | `opacity 0.45`, `cursor: not-allowed` |
| Selected entity | `primary.main` text/border, subtle `alpha(primary, 0.1)` fill |
| Live / connected link | `interactive.main` stroke + `interactive.glow` |
| Hover on a topology node | Related route highlighted, unrelated paths dimmed |
| Blocking action | Dialog with `Alert severity="warning" variant="outlined"`, single `Close` action |
| Destructive/stateful action | Confirmation dialog: `Cancel` (`color="inherit"`) + `Confirm` (`variant="contained"`) |
| Empty | Short uppercase `text.secondary` line, no illustration |
| Loading | Not present today — new async work uses an inline skeleton or a `—` placeholder, never a full-screen spinner |

---

## 6. Overlay taxonomy — pick the right one

| Need | Use | Precedent |
|---|---|---|
| Short confirmation / blocking warning | MUI `Dialog`, `maxWidth="xs"`, `fullWidth` | `PowerToggle` |
| Multi-field configuration | Tabbed `Dialog` | `SettingsDialog` (System / Satellite / Radio / Cellular) |
| Contextual detail for a topology node | Right-side drawer, bg `panel.drawer` | `ControlRoomDrawer` |
| Comparable/parallel live data | Draggable floating window (`Paper`, radius 8, header drag handle, close button) | `ModemCompareWindow`, `CameraWindow` |
| Transient notice | `NotificationTray`, top-right of the viewport | — |

Floating-window contract: pointer-drag from the header only, never from a `button` (`event.target.closest("button")` guard), clamped to the viewport, `setPointerCapture` on the header, position held in local state.

---

## 7. Charts (`MonitoringGraph`, Recharts)

- X axis: seconds. Y axis: Mbps. Optional right axis: ms latency.
- Series toggles via `SeriesCheckbox`, colored with theme palette colors — Upload, Download, Latency, plus a **dashed** Bandwidth ceiling line.
- Ticks `11–12px`, `text.secondary`. Grid uses `divider`.
- Never introduce a chart library other than Recharts.

---

## 8. Adding a feature — required sequence

1. **Place it.** Left sidebar = fleet/inventory. Viewport = spatial/topological. Right panel = the selected entity's controls & telemetry. Footer of the right panel = global mode/settings actions.
2. **Reuse first.** Search `src/components/GLOBAL/` before creating anything.
3. **Scaffold** `src/components/<Name>/{<Name>.tsx, <Name>.styles.ts, index.ts}`.
4. **Type the props**, export the interface, document non-obvious props with a one-line JSDoc (existing convention).
5. **Tokens only** — pull every color/space/size from `theme`.
6. **Status via `linkStatus.ts`.**
7. **Accessibility**: `aria-label` on every icon-only control, `role`/`aria-checked` on custom switches, `aria-busy` on async controls, `title` on matrix cells.
8. **Verify**: `bunx tsgo --noEmit`, then a preview screenshot at 1600×950 checking that nothing overflows and no text is under 12px.

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
  quality: number;
  /** Greys the readout when the link is powered off. */
  disabled?: boolean;
}

export function Example({ label, quality, disabled = false }: ExampleProps) {
  const theme = useTheme();
  const tone = theme.palette.status[qualityStatus(quality)];
  return (
    <ExampleRoot>
      {label}
      <ExampleValue tone={disabled ? theme.palette.text.disabled : tone}>
        {disabled ? "—" : `${quality}%`}
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

## 9. Forbidden

- `react-router-dom` or any router other than TanStack Router.
- A light theme, a theme switcher, or `prefers-color-scheme` branching.
- Hex/rgb literals, `text-white`-style utilities, Tailwind classes in components.
- New icon sets — MUI Icons only.
- Font sizes below `0.75rem`.
- `sx` for layout/structure.
- Full-screen loading spinners and modal stacking more than one deep.
- Re-enabling MUI ripple.

---

## 10. Visual reference index

| # | File | Shows |
|---|---|---|
| 01 | `screenshots/01-tactical-default.png` | Default three-column shell, tactical map |
| 02 | `screenshots/02-modem-expanded.png` | Right panel, modem channels expanded |
| 03 | `screenshots/03-vehicle-selected.png` | Selected-entity state across all three columns |
| 04 | `screenshots/04-settings-system-tab.png` | Tabbed dialog, System tab, PRECHECK/RUN |
| 05 | `screenshots/05-settings-radio-tab.png` | Tabbed dialog, field layout |
| 06 | `screenshots/06-logical-topology.png` | Logical mode, clusters → router/relay → CONTROL ROOM |
| 07 | `screenshots/07-control-room-drawer.png` | Right drawer pattern |
| 08 | `screenshots/08-camera-window.png` | Floating draggable window |
| 09 | `screenshots/09-confirm-power-dialog.png` | Confirmation dialog pattern |
| 10 | `screenshots/10-sidebar-collapsed.png` | 32px collapsed rail |
| 11 | `screenshots/11-tablet-768.png` | 768×1024 behavior |
| 12 | `screenshots/12-mobile-390.png` | 390×844 behavior |

Match new work to the closest precedent above before inventing a layout.

---

## 11. Handoff checklist

- [ ] Component folder with `.tsx` + `.styles.ts` + `index.ts`
- [ ] Exported, strictly typed props interface
- [ ] Zero hard-coded colors, spacing, or fonts
- [ ] Status colors derived from `linkStatus.ts` + `palette.status`
- [ ] Uppercase, letter-spaced labels; nothing under 12px; tabular numerals on metrics
- [ ] Correct overlay type per §6
- [ ] Off/disabled/selected/empty states handled per §5
- [ ] `aria-label` on every icon-only control
- [ ] Reused GLOBAL components wherever a pattern already exists
- [ ] `bunx tsgo --noEmit` clean
- [ ] Preview verified at 1600×950, 768×1024, 390×844
