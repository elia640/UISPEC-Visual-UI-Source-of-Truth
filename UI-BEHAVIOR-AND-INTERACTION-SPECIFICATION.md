# UI BEHAVIOR & INTERACTION SPECIFICATION

Companion to `UI-UX-Technical-Design-Specification-v2.md` (structure & visuals) and `UISPEC-v2.md` (visual source of truth).
This document describes **behavior only**: what every interactive element does, which state changes, which components react, and what happens next.

- Source of truth: the actual code in `src/` at the time of writing.
- Nothing here is invented. Anything not determinable from the code is marked `UNKNOWN / NEEDS VERIFICATION`.
- Application is **100% client-side with static in-memory data** (`src/data/*.ts`). There is **no backend, no API call, no persistence, no router navigation beyond the single route `/`**. Every "data change" below is React state in memory and is lost on reload.

---

## 0. GLOBAL BEHAVIORAL FACTS

| Fact | Detail |
|---|---|
| Routes | One: `/` (`src/routes/index.tsx`). No other route exists. No interaction changes the URL. |
| Data source | `src/data/network.ts`, `src/data/modems.ts`, `src/data/notifications.ts` — static constants. |
| Mutations | None of the static data is ever mutated. All toggles/edits live in local component state only. |
| Async | No `fetch`, no promises, no loading spinners anywhere. The only time-based behavior is the REBOOT countdown (`setInterval`, 1 s ticks). |
| Toasts | No toast/snackbar system is mounted. No success or error messages appear anywhere. |
| Errors | No error boundary UI, no validation error text, no failure paths. Invalid form input is silently ignored. |
| Persistence | None. No localStorage, sessionStorage, cookies or URL state. |
| Derived values | `src/lib/linkStatus.ts` (quality/rate/temperature/CPU/voltage → `good \| marginal \| poor`) and `src/lib/telemetry.ts` (`estimateLatency(quality)`). |

### Top-level state owner — `MonitorPage` (`src/routes/index.tsx`)

| State | Type | Initial | Written by | Read by |
|---|---|---|---|---|
| `linksOn` | boolean | `true` | Mesh Links button | TacticalMap (draws links + legend), LogicalTopology (colours edges) |
| `sidebarOpen` | boolean | `true` | Sidebar collapse / rail | FleetSidebar |
| `mode` | `"tactical" \| "logical"` | `"tactical"` | ViewModeSwitch (right-panel footer) | MonitorViewport |
| `selectedVehicleId` | string \| null | `null` | Sidebar row, map card, topology card, Control Room button, canvas click | Sidebar, map, topology, ControlRoomPanel |
| `selectedRelayId` | string \| null | `null` | Sidebar relay row, control-room marker click | Sidebar, map (recenter) |

**Mutual exclusivity rules coded in `index.tsx`:** selecting a relay clears the vehicle; selecting a vehicle *from the viewport* clears the relay; `onOpenControlRoom` clears both. Selecting a vehicle *from the sidebar* does **not** clear the relay (asymmetry — see Gaps §18).

---

## 1. CORE PRINCIPLE — worked example

**SETTINGS button**

1. User clicks `Settings` in the right-panel footer.
2. `settingsOpen` becomes `true` in `ControlRoomPanel`.
3. `SettingsDialog` mounts on the **System** tab (`tab` state initialises to `"system"`).
4. The Radio-frequency field is seeded once from the `radioFrequency` prop (`useState(String(radioFrequency))`); all other fields use hard-coded `defaultValue`s and are uncontrolled.
5. Tabs switch the visible body only; field values in other tabs stay mounted-then-unmounted (unmounted tabs lose uncontrolled edits — see Gaps).
6. Typing in the frequency field updates local string state only. No other component reacts.
7. `Apply` runs `Number.parseFloat(frequency)`.
8. If it is a number → `onRadioFrequencyChange(parsed)` updates `radioFrequency` in `ControlRoomPanel`; if `NaN` → nothing is written, no error is shown.
9. The dialog closes in both cases.
10. `Cancel`, ESC and backdrop click close without applying anything; the frequency string is **not** reset (the component stays mounted, so a re-open shows the last typed text).
11. No toast, no navigation, no request.

Every element below is documented at this level.

---

## 2. INTERACTION INVENTORY

**Buttons / icon buttons:** sidebar collapse, collapsed rail (expand), Mesh Links, notification bell, map zoom in / zoom out / reset view, camera icon (compact card), CAMERA button (full card), camera close, camera quality (H/M/L), Control Room button, Settings button, Apply, Cancel, dialog close, Run (precheck), REBOOT, coordinate dialog Cancel / Place, compare-window close, connectivity-map enlarge/shrink, channel RF-details expander, collapse chevrons (HALO panel, SIM group, card details), power-confirmation Confirm / Cancel / Close.
**Toggles / switches:** PowerToggle (HALO channels, radio, per-vehicle cellular/SATCOM/RADIO modems, per-SIM).
**Segmented control:** Tactical / Logical.
**Checkboxes:** chart series (Upload / Download / Latency / Bandwidth), compare-monitoring list, mesh-matrix metric (Modulation / SNR / RSSI — behaves as radio).
**Rows / cards / markers:** sidebar vehicle rows, sidebar relay rows, map platform cards, map control-room marker, map relay marker, topology platform cards, topology CONTROL ROOM node, mesh-matrix cells (tooltip only).
**Draggables:** map canvas (pan), map relay marker, map control-room marker, connectivity map panel, compare window.
**Overlays:** SettingsDialog, CameraWindow, CoordinateDialog, PowerToggle confirmation dialog, notification Popover, quality Menu, ModemCompareWindow (floating, non-modal).
**Tooltips:** every marker, chip, state chip, fault badge, health chip, matrix cell, compact card (full unit name).
**Not present anywhere:** search, filters, sorting, pagination, multi-select, drag-and-drop reordering, links/navigation items, textareas, radio inputs (native), form `submit` events.

---

## 3. INTERACTION MATRIX

| Element | Location | Trigger | Action | UI Result | State Change | Data Change | Navigation | Confirmation | Loading | Error |
|---|---|---|---|---|---|---|---|---|---|---|
| Collapse panel | Sidebar header | click | collapse sidebar | Sidebar → 32 px rail with vertical title | `sidebarOpen=false` | none | no | no | no | n/a |
| Collapsed rail | Left edge | click | expand sidebar | Full 296 px panel returns | `sidebarOpen=true` | none | no | no | no | n/a |
| Vehicle row | Sidebar › Vehicles | click | select / deselect platform | Row highlights; map recenters on unit; right panel re-binds | `selectedVehicleId` = id or `null` | none | no | no | no | n/a |
| Relay row | Sidebar › Relays | click | focus relay | Row highlights; map recenters on relay marker; vehicle selection cleared | `selectedRelayId`, `selectedVehicleId=null` | none | no | no | no | n/a |
| Metric checkbox | Sidebar › Link Matrix | click | switch matrix metric | Cell values and legend change; colour only in Modulation | `metric` (local) | none | no | no | no | n/a |
| Matrix cell | Sidebar › Link Matrix | hover | native title tooltip | `A ↔ B: value unit` | none | none | no | no | no | n/a |
| Mesh Links | Viewport top-right (tactical only) | click | toggle link overlay | Lines + legend appear/disappear | `linksOn` | none | no | no | no | n/a |
| Notification bell | Viewport top-right | click | open tray | Popover anchored bottom-right with 4 static alerts | `anchor` (local) | none | no | no | no | n/a |
| Zoom in / out | Map bottom-right | click | zoom ×1.3 / ÷1.3 about centre | Canvas scales, clamped 0.6–6 | `viewport` | none | no | no | no | n/a |
| Reset view | Map bottom-right | click | reset | zoom 1, offset 0,0 | `viewport` | none | no | no | no | n/a |
| Map canvas | Map | wheel | cursor-anchored zoom | Canvas scales | `viewport` | none | no | no | no | n/a |
| Map canvas | Map | left drag (>4 px) | pan | Canvas translates; the terminating click is swallowed | `viewport`, `panning` | none | no | no | no | n/a |
| Platform card (compact) | Map marker | click | select / deselect | Card gets selected border; map recenters; right panel re-binds | `selectedVehicleId`, `selectedRelayId=null` | none | no | no | no | n/a |
| Camera icon | Compact card | click (stopPropagation) | open camera | Full-screen portal modal | `cameraOpen` (per card) | none | no | no | no | n/a |
| Control-room marker | Map | click (not after drag) | show control-room parameters | Right panel title → CONTROL ROOM | `selectedVehicleId=null`, `selectedRelayId=null` | none | no | no | no | n/a |
| Control-room / relay marker | Map | drag | move marker | Marker follows pointer, clamped 2–98 % | marker `position` | none | no | no | no | n/a |
| Control-room / relay marker | Map | right-click | open coordinate dialog | Dialog with X/Y pre-filled | `coordTarget` | none | no | no | no | silent on NaN |
| CONTROL ROOM node | Logical view | click / Enter / Space | show control-room parameters | Right panel re-binds | both selections → `null` | none | no | no | no | n/a |
| Topology canvas background | Logical view | click | clear selection | Edges un-dim; right panel → CONTROL ROOM | both selections → `null` | none | no | no | no | n/a |
| Topology card | Logical view | hover | emphasise route | That route's edges thicken; others drop to 0.15 opacity | `hovered` | none | no | no | no | n/a |
| Topology card | Logical view | click | select / deselect | Selected border; right panel re-binds; route stays emphasised | `selectedVehicleId` | none | no | no | no | n/a |
| Connectivity map header | Logical view | drag | reposition panel | Panel follows pointer, clamped to window | `position` | none | no | no | no | n/a |
| Enlarge / shrink | Connectivity map | click | resize panel | Larger/smaller wheel | `expanded` | none | no | no | no | n/a |
| Wheel node | Connectivity map | hover | focus node | Non-incident edges dim to 0.15 | `focused` | none | no | no | no | n/a |
| HALO collapse chevron | Right panel | click | expand/collapse channels | Channel list + REBOOT row appear | `modemExpanded` | none | no | no | no | n/a |
| Channel name | Right panel › HALO | click | toggle RF details | SINR / RSSI / RSRP row | `openMetrics[id]` | none | no | no | no | n/a |
| PowerToggle | everywhere | click | request power change | Confirmation dialog opens | dialog `open`; on Confirm the owning boolean flips | none | no | **yes** | no | blocked-state alert |
| REBOOT | Right panel › HALO (expanded) | click | start 10 s cycle | Panel body dims; toggles disabled; button shows `REBOOTING Ns`, fills progress | `rebooting`, `remaining` | none | no | no | countdown (not a spinner) | n/a |
| Tactical / Logical | Right-panel footer | click | switch viewport | Map ⇄ topology; Mesh Links button appears only in tactical; connectivity map only in logical | `mode` | none | no (no URL change) | no | no | n/a |
| Control Room | Right-panel footer | click | show control room | Panel title → CONTROL ROOM; button gets active style | `selectedVehicleId=null` (relay id is **not** cleared) | none | no | no | no | n/a |
| Settings | Right-panel footer | click | open settings | Dialog on System tab | `settingsOpen` | none | no | no | no | n/a |
| Compare checkbox | Right panel (control-room context only) | click | add/remove compare window | Floating graph window appears/disappears, offset 32 px per index | `compared[]` | none | no | no | no | n/a |
| Compare window header | Floating window | drag | reposition | Window follows pointer, clamped | `pos` | none | no | no | no | n/a |
| Compare window close | Floating window | click | remove | Window unmounts; checkbox unchecks | `compared[]` | none | no | no | no | n/a |
| Series checkbox | Any monitoring graph | click | show/hide line | Line added/removed instantly (no animation) | `visible[key]` | none | no | no | no | n/a |
| Run (precheck) | Settings › System | click | invokes `onRunPrecheck?.()` | **Nothing** — no handler is passed from the page | none | none | no | no | no | n/a |
| Apply | Settings | click | parse + apply frequency | Dialog closes | `radioFrequency` if parseable | none | no | no | no | silently ignored on NaN |
| Cancel / ESC / backdrop | Settings | click / key | close | Dialog closes, nothing applied | `settingsOpen=false` | none | no | no | no | n/a |
| Quality H/M/L | Camera window header | click | open menu | MUI Menu, current value marked selected | `anchor`, then `quality` | none | no | no | no | n/a |
| Camera close / backdrop | Camera window | click | close | Modal unmounts; quality **persists** for that card | `cameraOpen=false` | none | no | no | no | n/a |
| Place | Coordinate dialog | click | move marker | Marker jumps to entered position (clamped 0–100) | marker `position` | none | no | no | no | silent on NaN |

---

## 4. BUTTON SPECIFICATION

### 4.1 Collapse panel / Collapsed rail
Component `IconButton` / `CollapsedRail`. Always enabled. Click → toggles `sidebarOpen`. Sidebar content unmounts when collapsed; all sidebar-local state (matrix metric) resets on re-expand. No confirmation, no loading, no navigation.

### 4.2 Mesh Links (tactical only)
`LinksButton`, `aria-pressed={linksOn}`, active styling when on. Click → `linksOn = !linksOn`.
→ Map: all link lines hide; the status legend (bottom-left) hides with them.
→ With links off, hovering a vehicle marker still reveals **only that vehicle's** links (`showsLink` uses `hoveredId`).
→ Logical view: `linksOn=false` renders every edge in `divider` grey instead of status colour. The button is not rendered in logical mode, so the state can only be changed from the tactical view.

### 4.3 Zoom in / Zoom out / Reset view
Zoom multiplies by 1.3 / 0.3846 about the container centre, clamped `[0.6, 6]`. Reset returns `{zoom:1,x:0,y:0}`. Never disabled — clicking at the clamp limit is a no-op. Marker positions are unaffected (they are percentage-based).

### 4.4 REBOOT (`GLOBAL/RebootButton`, used by `ModemPanel`)
```text
Button: REBOOT
Enabled when: HALO panel is expanded and no reboot is running
On Click:
→ remaining = 10, rebooting = true
→ onRebootingChange(true) → ModemPanel sets rebooting=true
→ DimWrap dims header, meter, health row and channel list
→ every channel PowerToggle receives disabled=true
→ button is disabled, aria-busy=true, label "REBOOTING 10s", progress fill grows
→ 1 s interval decrements to 0
→ at 0: rebooting=false, onComplete() → optional onReboot(modem.id) (no handler is wired today)
→ panel un-dims, toggles re-enable, label returns to "REBOOT"
→ No toast, no navigation, no request, no data change
Re-click: possible only after the cycle finishes.
```

### 4.5 Control Room (right-panel footer)
`active` styling when no vehicle is selected. Click → `onSelectVehicle(null)` only. The panel re-binds to `controlRoomModem` / `controlRoomRadio`, the monitoring graph loses its latency series (`withLatency={vehicle!==null}`), and the COMPARE HALO MONITORING checkbox row appears. Note: it does **not** clear `selectedRelayId`.

### 4.6 Settings
See §1. Enabled always.

### 4.7 Run (Settings › System › Precheck)
Calls `onRunPrecheck?.()`. `ControlRoomPanel` accepts the prop but `src/routes/index.tsx` never passes it → **the button does nothing today.** No visual feedback of any kind. Documented gap.

### 4.8 Apply / Cancel (Settings)
Apply: parse frequency → apply if numeric → close. Cancel: close only. Neither validates any other field; satellite/radio/cellular fields are uncontrolled `defaultValue` inputs whose values are **never read** — editing them has no effect on anything.

### 4.9 Camera controls
`CameraIconButton` (compact card) and `CAMERA` (full card) set `cameraOpen=true` for that card instance. The compact-card handler calls `event.stopPropagation()` so opening the camera does not select the platform. Close button and backdrop click both close; ESC does **not** (no key handler, it is a hand-rolled portal, not an MUI Dialog).

### 4.10 Confirmation dialog buttons (PowerToggle)
`Confirm` (contained) applies `onChange(!checked)` and closes; `Cancel` closes only. In the blocked case only a single `Close` button is rendered — there is no way to force the change.

### 4.11 Coordinate dialog: Cancel / Place
Place parses both fields; if either is `NaN` the handler returns early — the dialog **stays open** and shows nothing. Otherwise the position is clamped 0–100 and applied, and the dialog closes.

### 4.12 Compare-window close / connectivity enlarge-shrink / notification bell
All are pure local-state toggles with no side effects beyond their own component.

---

## 5. FORM FIELD SPECIFICATION

Only two dialogs contain fields. **There is no `<form>` element, no submit event, and no validation library anywhere in the app.**

### 5.1 Settings › Radio › Radio frequency (MHz)
| Property | Value |
|---|---|
| Component | `SettingsField` (MUI TextField, outlined, small) |
| Type | `number`, `step 0.025`, `min 30`, `max 6000` (browser hints only — not enforced in code) |
| Default / source | `String(radioFrequency)` from `ControlRoomPanel` state, seeded **once** at mount (initial `2412`) |
| Required | Effectively yes for Apply, but empty input is simply ignored |
| On change | Updates local string state only. Nothing else reacts. |
| On blur / focus | No handlers |
| Validation | `Number.parseFloat` at Apply time only. `NaN` → skip. |
| Error message | None |
| Disabled / read-only | Never |

### 5.2 Settings — uncontrolled fields (no state, no readers)
Satellite: Satellite (select TELS-1 / TELS-2, default TELS-1), Beam/Transponder (text, `KA-04`), Symbol rate (number, `12`), Modulation (select QPSK / 8PSK / 16APSK, default 8PSK).
Radio: Channel bandwidth (select 5/10/20 MHz, default 20), Mesh network ID (text, `MESH-A`), TX power (number, `27`).
Cellular: Preferred network (Auto / LTE / 5G NR, default 5G), APN (text, `fleet.ops`), SIM priority (SIM 1–3, default SIM 1), Data cap per SIM (number, `50`).
**Behavior for all of the above:** on change the DOM value changes and nothing else; Apply ignores them; switching tabs unmounts the body, so edits are lost. This is presentational scaffolding.

### 5.3 Coordinate dialog — EAST (X) / NORTH (Y)
Controlled strings, re-seeded from the live marker position each time `open` flips to true (`useEffect`). Change → local state only. Place → parse, clamp `[0,100]`, apply, close. `NaN` → silent no-op, dialog stays open. Hint text: "Enter the grid position, 0–100 on each axis."

---

## 6. FORM SUBMISSION FLOWS

```text
SETTINGS
Open (Settings button)
↓ tab = "system"; frequency = current radioFrequency (seeded once)
User edits fields
↓ only the frequency field is read
Apply
↓ Number.parseFloat
valid → onRadioFrequencyChange(value) → dialog closes
invalid → nothing applied → dialog closes anyway
No loading, no toast, no refresh, no error surface.
Cancel / ESC / backdrop → close, discard.
Values are NOT reset afterwards (component stays mounted).
```

```text
COORDINATE ENTRY
Right-click marker → dialog opens, fields seeded from live position
Edit X / Y
Place
↓ parse
valid → clamp 0–100 → marker jumps → dialog closes
invalid → dialog STAYS OPEN, no message, no highlight
Cancel / ESC / backdrop → close, marker unchanged
```

No other flow in the app has a submit step.

---

## 7. MODAL / DIALOG BEHAVIOR

| Modal | Opened by | Initial state | ESC | Backdrop | Close btn | Cancel | Submit | Underlying screen |
|---|---|---|---|---|---|---|---|---|
| SettingsDialog | Settings button | System tab; frequency seeded; other fields at defaults | closes (MUI `onClose`) | closes | closes | closes, discards | Apply parses + applies frequency, closes | inert, dimmed by MUI backdrop |
| PowerToggle confirmation | any PowerToggle click | Title `CONFIRM POWER CHANGE`, or `ACTION BLOCKED` when it is the last active link | closes | closes | n/a | closes, no change | Confirm flips the boolean, closes | inert |
| CoordinateDialog | right-click relay / control-room marker | X/Y from live marker position | closes | closes | n/a | closes | Place applies or (on NaN) keeps the dialog open | inert |
| CameraWindow | camera icon / CAMERA button | quality `H`; menu closed | **does not close** (no key handler) | closes | closes | n/a | n/a | inert behind a hand-rolled backdrop, rendered via `createPortal` to `document.body` |
| Notification Popover | bell | anchored bottom-right of the bell | closes | closes (MUI popover) | n/a | n/a | n/a | interactive elsewhere |
| ModemCompareWindow | compare checkbox | positioned at `120+32*index` | no effect | no backdrop (non-modal) | removes window + unchecks | n/a | n/a | **fully interactive** — multiple windows can stay open |

Closing behavior summary: SettingsDialog **discards** (except an already-applied frequency); CoordinateDialog **discards**; CameraWindow **preserves** its chosen quality for that card; the confirmation dialog **applies only via Confirm**.

MUI dialogs supply focus trapping and focus restoration to the trigger. `CameraWindow` and `ModemCompareWindow` are not MUI dialogs and therefore have **no focus trap, no focus restoration, no ESC** — documented gap.

---

## 8. DROPDOWNS / MENUS

### 8.1 Camera quality menu
```text
Click H / M / L control in the camera header
→ MUI Menu opens anchored to the button
→ Current quality row is rendered `selected`
→ Arrow keys / Home / End move focus (MUI default); ESC closes without change
→ Click or Enter on a value
→ Menu closes
→ quality state changes (H | M | L)
→ Header button label changes
→ Footer right-hand text becomes "QUALITY <value>"
→ The feed image itself does NOT change (single static asset — simulation only)
```

### 8.2 Settings selects (Satellite, Modulation, Channel bandwidth, Preferred network, SIM priority)
Standard MUI select menus, uncontrolled. Selecting an option changes the displayed value and nothing else — no reader consumes it.

### 8.3 Tactical / Logical segmented control
`ToggleButtonGroup exclusive`. Clicking the already-active option yields `null` and is ignored (`next && onModeChange(next)`), so the mode can never be cleared. Keyboard: Tab into the group, arrow keys move between buttons, Enter/Space activates.

There are no context menus other than the right-click coordinate entry, and no autocomplete anywhere.

---

## 9. TABLES / LISTS

### 9.1 Sidebar › Vehicles (5 rows, static)
Columns: Platform / Range (active link kinds joined with ` + `) / Quality (`QualityMeter`).
Row click → toggle select: `selectedVehicleId === id ? null : id`.
Effects: row highlight, **map recenters on that unit** (`centerOn` effect in TacticalMap), map card gains selected border, right panel title and all its panels re-bind, monitoring graph switches to a vehicle seed and gains the latency series, the compare checkbox row disappears.
Clicking the selected row again deselects → right panel returns to CONTROL ROOM. No multi-select. No row actions, no context menu, no sort/filter.
`role="button"`, `aria-label="Select <label>"`. Rows are not focusable (`tabIndex` is absent) — keyboard users cannot reach them. Gap.

### 9.2 Sidebar › Relays (1 row)
Toggle select as above; additionally clears `selectedVehicleId`. Effect: map recenters on the relay marker's **current dragged position**. Aria label: `Center map on <label>`.

### 9.3 Sidebar › Link Matrix (`MeshMatrix`)
Square N×N table (CR + 5 platforms + relay). Cells are read-only; the diagonal shows `—`. Hover gives a native `title` tooltip. Metric checkboxes act as a radio group (`checked={metric===id}`), switching between dB margin / SNR / RSSI. Colour fills only in Modulation mode; the legend swaps to "values in <unit>" for the other two. Missing pairs fall back to a margin of 10.

### 9.4 Notification list
Four static items, severity-coloured, non-clickable. No read/unread, no dismiss, no clear-all. The badge count is always `items.length` (4) and never decreases.

---

## 10. FILTERS AND SEARCH

**None exist.** There is no search input, no filter control, no sort control and no pagination anywhere in the application. The closest behaviors are:

| Pseudo-filter | Mechanism | Notes |
|---|---|---|
| Mesh Links toggle | boolean | hides/shows all link overlays |
| Hover reveal (links off) | `hoveredId` | shows only the hovered vehicle's links |
| Route emphasis (logical) | `hovered ?? selectedVehicleId` | non-matching edges drop to 0.15 opacity; cards dim |
| Chart series checkboxes | `visible` record | filters plotted lines |
| Matrix metric | `metric` | changes the value set displayed |

None debounce (no text input exists), none persist, none have an empty-result state.

---

## 11. NAVIGATION

| Current Location | User Action | Destination | State Preserved | URL Change |
|---|---|---|---|---|
| `/` | Any interaction in the app | `/` | yes | **no** |
| `/` tactical | Click `Logical` | `/` with `mode="logical"` | selections, links, sidebar all preserved | no |
| `/` logical | Click `Tactical` | `/` with `mode="tactical"` | preserved | no |
| `/` | Select a platform | `/` | right panel re-binds | no |
| `/` | Open any dialog | `/` | underlying state untouched | no |

Explicitly non-navigating: selecting a platform, selecting a relay, opening Settings, opening the camera, switching view mode, clicking the CONTROL ROOM node, dragging markers. **The application has exactly one URL and no history entries are ever pushed.**

Note: mode is owned by `MonitorPage` but the control lives in the right panel footer; switching mode unmounts the previous view entirely, so map zoom/pan, marker drags, hovered state, HALO channel toggles inside the viewport cards and camera-window state in viewport cards **all reset** when returning.

---

## 12. STATE TRANSITIONS

### PowerToggle
```text
OFF/ON (idle)
  ↓ click
CONFIRMING            (dialog "CONFIRM POWER CHANGE")
  ↓ Confirm            ↓ Cancel / ESC / backdrop
TOGGLED (opposite)    idle (unchanged)
```
```text
ON and lastActive=true
  ↓ click
BLOCKED               (dialog "ACTION BLOCKED", warning Alert, only a Close button)
  ↓ Close
ON (unchanged — the change can never be forced)
```
`lastActive` is computed per context: HALO channels `on && activeCount<=1`; vehicle modems `isLastLink` over cellular/SATCOM/radio; SIMs `activeSims<=1`. When `confirm={false}` (not used today) the toggle applies immediately and silently refuses when blocked.

### RebootButton
```text
IDLE
  ↓ click
REBOOTING (10 → 1, button disabled, panel dimmed, toggles disabled)
  ↓ tick reaches 0
COMPLETE → onComplete() → IDLE
```
No cancel path exists once started.

### Platform selection (global)
```text
NONE (right panel = CONTROL ROOM)
  ↓ click row / map card / topology card
SELECTED <id>  (map recenters, panel re-binds, compare row hidden)
  ↓ click the same element again          ↓ click Control Room / CONTROL ROOM node / topology background
NONE                                      NONE
```

### Map viewport
```text
IDLE → (pointerdown) PRESSED → (>4 px move) PANNING → (pointerup) IDLE + click swallowed
IDLE → (pointerdown, <4 px, pointerup) → click delivered to the card underneath
```

### CameraWindow
```text
CLOSED → (camera button) OPEN, quality=H (or last chosen for that card)
OPEN → (quality button) MENU OPEN → (pick) OPEN with new quality
OPEN → (close / backdrop) CLOSED   [ESC does nothing]
```

---

## 13. DATA FLOW

```text
static data (src/data/*.ts)
      ↓ imported directly by components (no store, no context, no fetch)
React local state (page-level for selection/mode/links, component-level for everything else)
      ↓ derived by src/lib/linkStatus.ts + src/lib/telemetry.ts
UI (colours, chips, bars, edges, labels)
```

| Feature | Data source | State owner | Mutates | Affects | Persistence |
|---|---|---|---|---|---|
| Platform / relay list | `data/network.ts` | — (constant) | never | sidebar, map, topology, matrix, wheel | none |
| Selection | — | `MonitorPage` | `selectedVehicleId`, `selectedRelayId` | sidebar rows, map recenter + card border, topology dimming, right panel binding, graph seed | none |
| View mode | — | `MonitorPage` | `mode` | viewport contents, Mesh Links button, connectivity map | none |
| HALO channel power | `data/modems.ts` | `ModemPanel.off` | local record | state chip, signal bars, rate cell, `lastActive` computation | none |
| Vehicle modem / SIM power | `data/network.ts` | `PlatformCard` | `cellularOn`/`satcomOn`/`radioOn`/`sims` | bars, ON/OFF text, redundancy blocking | none, and **not** reflected on the map links |
| Marker positions | `GROUND_STATION_POSITION`, `relay.x/y` | `useMapDrag` per marker | `position` | link endpoints, off-screen arrow, coordinate dialog, relay recentering | none |
| Radio frequency | initial `2412` | `ControlRoomPanel` | `radioFrequency` | passed back into the dialog only — **displayed nowhere else** | none |
| Compare windows | `monitoringSamples(seed)` | `ControlRoomPanel.compared` | array of ids | floating graph windows | none |
| Monitoring samples | `monitoringSamples(seed)` in `data/modems.ts` | derived via `useMemo` | — | graph lines | none |
| Alerts / fault badges | `data/notifications.ts`; `alertCountFor(label)` counts non-`good` alerts whose title starts with the unit label | — | never | bell badge (always 4), per-card fault badge | none |

**No API endpoints exist. Do not invent any.** Latency values are computed, not measured (`estimateLatency`).

---

## 14. LOADING / ERROR / EMPTY

### Loading
The only "busy" state in the entire application is the reboot cycle:
- Disabled: the REBOOT button itself, every channel PowerToggle in that HALO panel.
- Dimmed: the HALO panel body (`DimWrap`).
- Indicator: in-button countdown text plus a progress fill; **no spinner, no skeleton, no MUI progress component**.
- The rest of the app stays fully interactive during the countdown.
- Charts render instantly from in-memory arrays; there is no first-paint loading state (charts may appear empty for one frame before the ResponsiveContainer measures — known cosmetic debt).

### Error
No error UI exists. Specifically: no error boundary, no field error text, no failure toast, no retry affordance, no rollback logic. The two parse failures (`Apply`, `Place`) fail silently. The only warning-style surface in the app is the `ACTION BLOCKED` Alert inside the PowerToggle dialog.

### Empty
No empty states are implemented because every list is a non-empty static constant. `HealthMetrics` returns `null` when no metric is supplied; the topology renders nothing if `platforms` were empty (untested path). There is no CTA anywhere for an empty state. `UNKNOWN / NEEDS VERIFICATION`: intended empty-state design for future data-driven lists.

---

## 15. KEYBOARD / ACCESSIBILITY BEHAVIOR

| Component | Tab | Enter / Space | ESC | Arrows | ARIA |
|---|---|---|---|---|---|
| Buttons / IconButtons | focusable | activate | — | — | `aria-label` on every icon-only control |
| Mesh Links | focusable | activate | — | — | `aria-pressed={linksOn}` |
| PowerToggle | focusable (it is a styled `button`) | activate → opens dialog | closes dialog | — | `role="switch"`, `aria-checked`, `aria-label` |
| CollapseButton | focusable | toggle | — | — | `aria-expanded`, `aria-label="Toggle <region>"` |
| Channel name button | focusable | toggle RF row | — | — | `aria-expanded`, `aria-label="<ch> RF details"` |
| ViewModeSwitch | group focusable | activate | — | move between options | `aria-label="View mode"` per option |
| Checkboxes | focusable | Space toggles | — | — | `aria-label` via `slotProps.input` |
| MUI dialogs | trapped | — | closes | — | `aria-labelledby` on Settings |
| Sidebar rows | **not focusable** | — | — | — | `role="button"` + `aria-label` (role without tabIndex — gap) |
| Map platform cards | **not focusable** | — | — | — | `role="button"` when selectable |
| CONTROL ROOM node (logical) | focusable (`tabIndex=0`) | both Enter and Space open the panel | — | — | `aria-label` |
| Map markers | not focusable | — | — | — | `role="button"`, drag is pointer-only |
| CameraWindow | **no trap, no restore** | — | **no effect** | — | `role="dialog"`, `aria-label="<unit> camera feed"` |
| ModemCompareWindow | close button focusable | activate | no effect | — | `aria-label="Close <title>"` |
| QualityBar / QualityMeter | — | — | — | — | `role="progressbar"` with `aria-valuenow/min/max` |
| Map pan / zoom | — | — | — | **no keyboard equivalent** | zoom buttons are the accessible path |

Focus after modal close: MUI restores to the trigger; the portal-based CameraWindow and the floating compare window do not.

---

## 16. CROSS-COMPONENT EFFECTS

```text
Select platform (sidebar row | map card | topology card)
→ MonitorPage.selectedVehicleId = id
→ (from viewport only) selectedRelayId = null
→ FleetSidebar: that row renders selected
→ TacticalMap: useEffect centerOn(unit.x, unit.y); card border + raised z-index
→ LogicalTopology: activeRoute = id → that route's edges thicken to 2.4, all other edges drop to 0.15, other cards dim
→ ControlRoomPanel: title = unit.label; modem = vehicleModem(id); radio = vehicleRadio(id);
   samples = monitoringSamples(label.length + quality/10); graph gains the latency axis
→ ControlRoomPanel: the COMPARE HALO MONITORING checkbox row is hidden (control-room context only)
→ Already-open compare windows STAY OPEN (they are not cleared on selection) — see Gaps
→ Control Room footer button loses its active styling
```

```text
Select relay (sidebar)
→ selectedRelayId = id, selectedVehicleId = null
→ TacticalMap centers on the relay marker's current (possibly dragged) position
→ Right panel falls back to CONTROL ROOM (because the vehicle was cleared)
```

```text
Click CONTROL ROOM node (logical) | topology background | control-room map marker
→ both selections = null
→ right panel = CONTROL ROOM context, compare row reappears, latency series disappears
→ topology un-dims every edge and card
```

```text
Toggle Mesh Links
→ TacticalMap: link <g> hidden, legend hidden
→ LogicalTopology: edges rendered in divider grey (colour returns when on)
```

```text
Switch view mode
→ viewport unmounts the previous view: map zoom/pan, marker drags, hover, and card-local toggles are LOST
→ Mesh Links button mounts (tactical) / unmounts (logical)
→ ConnectivityWheel mounts (logical) / unmounts (logical→tactical), losing its position and expanded state
→ Selections, sidebar state and right-panel state survive
```

```text
Toggle a HALO channel off
→ StateChip shows NO CONN, SignalBars grey out, rate cell shows "—"
→ activeCount decreases → the remaining ON channel becomes lastActive → its next click is BLOCKED
→ The panel quality meter and RX rate are static and do NOT recalculate (documented gap)
```

```text
Drag the control-room marker off screen
→ OffscreenArrow appears at the map border, rotated toward the marker, labelled "CR"
→ Recomputed on marker move, pan, zoom and window resize
```

---

## 17. EXISTING UX FLOWS

1. **Inspect a platform** — click a sidebar row (or map/topology card) → map recenters, right panel re-binds → expand the HALO panel → expand a channel for SINR/RSSI/RSRP → click the row again (or Control Room) to return.
2. **Collapse / expand the sidebar** — click the chevron; click the rail to restore. Matrix metric resets to Modulation.
3. **Change power** — click any PowerToggle → confirm → boolean flips. On the last active link the dialog blocks instead.
4. **Reboot** — expand HALO → REBOOT → 10 s dimmed countdown → controls return.
5. **Open settings** — Settings → System tab → optionally Radio tab → edit frequency → Apply (only the frequency is read) or Cancel.
6. **Open a camera** — camera icon on a map/topology card → portal modal with the simulated road feed → close via the X or the backdrop.
7. **Change camera quality** — H/M/L → menu → pick → header + footer update; the image does not.
8. **Open notifications** — bell → popover with four static alerts → click outside to close. The badge never clears.
9. **Compare monitoring** — with no platform selected, tick entries in COMPARE HALO MONITORING → a draggable graph window appears per entry (offset 32 px each) → drag by the header → close via X (which also unticks).
10. **Map interaction** — wheel to zoom about the cursor, drag to pan (>4 px), zoom buttons, reset view, drag the relay and control-room markers, right-click a marker for coordinate entry, hover a vehicle to reveal its links when Mesh Links is off.
11. **Logical topology interaction** — hover a card to emphasise its route, click to select, click the CONTROL ROOM node or the empty canvas to reset, read the aggregated `↓ N Mbps` labels; relayed platforms route through a peer with a dashed segment and their traffic is summed into the peer's single trunk.
12. **Mesh links toggle** — see §4.2.
13. **Coordinate entry** — right-click the relay or the control room → type X/Y → Place.
14. **Connectivity map (logical)** — drag the header to reposition, enlarge/shrink, hover a node to isolate its edges.
15. **Link matrix** — switch metric between Modulation / SNR / RSSI, hover cells for the pair readout.

---

## 18. EXISTING BEHAVIOR GAPS

| Feature | Current behavior | Expected / needed | Status | Recommendation |
|---|---|---|---|---|
| Precheck `Run` | Handler is never passed from the page — the button is inert | Should run a visible precheck sequence with progress and result | Broken | Wire `onRunPrecheck` from `MonitorPage`; model progress on the REBOOT countdown pattern |
| API / async layer | Does not exist; all data is static | Needed for any real feature | Missing | Introduce loading/error vocabulary before the first async feature; nothing exists to reuse today |
| Error handling | No error UI at all; parse failures are silent | Inline field errors + a non-blocking failure surface | Missing | Add a single toast/snackbar host and inline `helperText` before adding forms |
| Form validation | None; `Apply` swallows `NaN` | Validate on blur + submit, block submit, show a message | Missing | Keep the field open on failure (the coordinate dialog already stays open — align both) |
| Settings fields | Satellite / Radio-extra / Cellular fields are uncontrolled and never read | Should be controlled and applied | Incomplete | Lift into `ControlRoomPanel` state alongside `radioFrequency` |
| Settings tab state | Uncontrolled tab bodies unmount, discarding edits | Values should survive tab switches | Bug-prone | Control every field; render tab bodies from one form state object |
| Radio frequency result | Applied to state but displayed nowhere | Should be visible in the radio panel / mesh info | Incomplete | Surface it in `RadioPanel` or the matrix frequency row |
| Card power toggles | Change local card state only; map links, quality bars and rates do not react | Turning a link off should change the topology and status | Incomplete | Lift per-unit link state to the page level |
| HALO panel metrics | Panel quality/rate stay static when channels are turned off | Should recompute | Incomplete | Derive panel values from active channels |
| Sidebar / card keyboard access | `role="button"` without `tabIndex` — unreachable by keyboard | Focusable and Enter/Space activatable | A11y bug | Add `tabIndex={0}` + `onKeyDown`, as the topology CONTROL ROOM node already does |
| CameraWindow | No ESC, no focus trap, no focus restore | Standard modal semantics | A11y bug | Rebuild on MUI `Dialog`, or add a key handler + focus management |
| Notifications | Static, count never clears, items not actionable | Read/dismiss/clear-all, deep-link to the unit | Missing | Model dismissal as page-level state |
| Compare windows | Survive a platform selection although the compare UI is control-room-only | Should close or be re-scoped on selection | Inconsistent | Clear `compared` when a vehicle is selected |
| Control Room button | Clears the vehicle but not the relay | Should clear both like the other reset paths | Inconsistent | Pass an `onOpenControlRoom` reset instead |
| Unused components | `ControlRoomDrawer`, `CommsAssetCard`, `CommsLinkRow`, `ThroughputChart` are exported but never rendered | — | Dead code | Delete or adopt; do not treat as live patterns |
| Route metadata | Title still reads "Comms Network Monitor" while the UI uses HALO naming | Consistent naming | Inconsistent | Update `head()` in `src/routes/index.tsx` |
| Persistence | Nothing survives reload | Sidebar state, mode and selection would benefit | Missing | Use URL search params (TanStack Router) rather than localStorage |
| Mobile | Fixed three-column shell overflows horizontally | Responsive behavior | Known debt | Out of scope for behavior work; see the design spec |

---

## 19. RULES FOR NEW FEATURES

1. Every interactive control must have a documented action in this file before it ships.
2. Every state-changing action must name its resulting state and every component that re-renders because of it.
3. Every async operation must define loading, success and error behavior — none exists today, so define it explicitly rather than assuming a pattern.
4. Every form must define validation, validation timing, error placement and what happens to the dialog on failure.
5. Every destructive or service-affecting action must use the existing `PowerToggle` confirmation pattern (`CONFIRM …` / `ACTION BLOCKED` + Alert).
6. Every MUI-based modal must support ESC, backdrop close and an explicit Cancel; state a discard/apply policy.
7. Reuse existing interaction patterns: selection = toggle-on-second-click; expansion = `CollapseButton` with `aria-expanded`; busy = the `RebootButton` countdown; series filtering = `SeriesCheckbox`.
8. Do not invent new interaction patterns when one already exists.
9. Keep shared cross-view state (selection, mode, links) in `MonitorPage`; keep purely local state in the component.
10. Do not invent API endpoints, persistence or toasts. If a feature needs them, add the primitive deliberately and document it here first.
11. Redundancy rule is non-negotiable: the operator must never be able to switch off the last active communication link — compute `lastActive` and pass it to the toggle.
12. Derive all status colours from `src/lib/linkStatus.ts`; never hard-code a threshold.
13. New interactive elements need an accessible name, a keyboard path, and a `role` consistent with §15.

---

## 20. FEATURE IMPLEMENTATION TEMPLATE

```markdown
# New Feature Behavior Specification

## Feature
<name and one-sentence purpose>

## Entry Point
<which existing control opens it, and where it lives>

## User Flow
1.
2.
3.

## Interactive Elements
| Element | Trigger | Behavior | Result |
|---|---|---|---|

## Fields
| Field | Input | Validation | On Change | On Submit |
|---|---|---|---|---|

## Buttons
| Button | Action | Confirmation | Loading | Success | Error |
|---|---|---|---|---|---|

## State Machine
```text
IDLE
  ↓ <trigger>
<STATE>
  ↓ <event>
<STATE>
```

## Data Flow
```text
UI → State (owner: <component>) → Data (<static | new source>) → UI (<components affected>)
```

## Loading
<what dims, what disables, what indicator, can the user work elsewhere>

## Empty
<what is shown, is there a CTA, what resolves it>

## Error
<where shown, retry, rollback, does the dialog stay open>

## Success
<final UI state, does anything close, is anything refreshed>

## Navigation
<state it explicitly if the feature does NOT navigate — the default in this app>

## Keyboard / Accessibility
<tab order, Enter/Space, ESC, focus target, focus return, aria attributes>

## Cross-component effects
<every other component that re-renders or re-binds>
```

---

## 21. VERIFICATION NOTES

- All behavior above was read from the current source in `src/`; runtime behavior was previously confirmed via Playwright captures in `screenshots-v2/`.
- Marked unknowns: intended empty-state design for future data-driven lists; the intended precheck sequence behind `onRunPrecheck`; whether the unused components (`ControlRoomDrawer`, `CommsAssetCard`, `CommsLinkRow`, `ThroughputChart`) are planned or abandoned.
- No application code was changed while producing this document.
