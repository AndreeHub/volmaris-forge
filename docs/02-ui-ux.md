# Volmaris Forge — UI/UX Build Guide for an LLM

> **Purpose.** This document tells an LLM (and any human reviewer) exactly how to design and build a Volmaris Forge–class UI: a modern, professional 2D CAD/schematic editor that beats the AutoCAD/Revit/Dendrit incumbents on *feel* while staying inside hard technical constraints (C++20 + Qt 6 + custom OpenGL viewport + custom ribbon, GPL-3.0 compatible assets only).
>
> Treat every value in this doc as a **hard spec**, not a suggestion. If you can't satisfy a constraint, surface the conflict — don't invent your way around it.

> **Implementation note (read first).** This guide was authored against an HTML/React prototype (see §13's JSX file list). The real product is **C++20 + Qt 6**. The impl agent must **translate** every JSX reference into the Qt equivalent: token module → C++ `Theme` singleton; `<canvas>` viewport → `QOpenGLWidget`; React components → `QWidget` subclasses; `prefers-reduced-motion` → `QStyleHints::motionPreference()`; etc. **Tokens, sizes, colors, shortcuts, interaction patterns are normative and unchanged.** The file layout in §13 describes the *prototype*, not the Qt build target — the Qt repo layout is in [`01-architecture.md`](01-architecture.md) §Repository Layout.

---

## 0. Operating principles for the LLM

Before generating anything, internalize these. They are the difference between a good output and slop.

1. **Density without noise.** This is an 8-hour-a-day engineering tool. Information density is a *virtue*; visual clutter is a *vice*. Every pixel should earn its place. If a label, icon, badge, or shadow doesn't communicate something a working engineer needs, delete it.
2. **Type and number first.** Engineers read coordinates, dimensions, and entity counts constantly. Numerics are **always** monospace. UI labels are sans-serif with tight tracking. Never use display fonts.
3. **Surfaces, not gradients.** Flat surfaces separated by 1px hairlines. Reserve shadow for floating overlays only (tooltips, popovers, modals). No gradients on chrome. No glass blur.
4. **One accent, used sparingly.** The accent color marks "this is selected" or "this is the current focus." If three things on screen are accented, two of them are wrong.
5. **Geometry on a true viewport.** The drawing canvas is an OpenGL surface in production. Its appearance must match: pure background, sub-pixel-aligned grid, crisp ink, no anti-aliasing artifacts from CSS hacks.
6. **No web tropes.** No hero sections, no card grids with rounded-2xl + shadow-xl, no emoji, no gradient meshes, no "AI-generated" iconography. This is a Qt desktop app, not a SaaS landing page.
7. **Don't invent features.** Stay inside the Phase 0–7 scope. Future panels (MEP browser, schedules, calc panels) get a *placeholder dock strip* and nothing more.

---

## 1. Design principles (project-level)

These are the 5 bullets that should appear at the top of any spec doc.

1. **Modern execution, Revit structure.** Ribbon + dockable palettes + status bar. But typography, spacing, color, motion are 2025-modern — not Office-2007.
2. **The chrome gets out of the way.** The viewport is the product. Chrome compresses to the minimum that supports the active task. Inactive UI fades back; active UI sharpens.
3. **Both themes are first-class.** Engineers work in offices (light) and homes (dark). Every component is designed in both themes simultaneously. Dark is *not* a tinted light theme.
4. **Keyboard-first, mouse-comfortable.** Every tool has a single-key shortcut. Modifiers are conventional (Ctrl, Shift, Alt). No command-line prompt. No `LINE↵` typing.
5. **Precision is visible.** Coordinates, units, scale, and active snap are always on screen in the status bar. Snap markers are color-coded and labeled.

---

## 2. Theme tokens (exact values)

Both themes are defined as a single token shape so the LLM can flip `mode` and get a coherent palette.

### 2.1 Palette — Light theme

```
Surfaces
  bg0  #EDEEF0   Window background / behind ribbon (slight cool tint)
  bg1  #F7F8F9   Ribbon, status bar
  bg2  #FFFFFF   Viewport canvas (pure white)
  bg3  #F2F3F5   Palettes, dock strips
  bg4  #E7E9EC   Hover
  bg5  #DCDFE3   Pressed / active group

Text
  tx0  #15171A   Primary (body, button labels)
  tx1  #52555B   Secondary (group labels, sub-text)
  tx2  #82868D   Tertiary (status numerics, hints)
  tx3  #A6AAB1   Hint / disabled

Lines
  bd0  #E4E6EA   Subtle divider (group separators, status bar segments)
  bd1  #D4D7DC   Standard border (panel edges)
  bd2  #B6BAC1   Strong border (focused input)

Accent (Forge Blue, default)
  accent       #2466E0
  accentSoft   #E8EFFD   (button background when active)
  accentText   #FFFFFF   (text on accent)

Viewport ink
  gridMinor    #EEF0F3   (100mm lines)
  gridMajor    #DDE1E6   (500mm lines)
  gridAxis     #BFC5CD   (origin axes)
  ink          #1B1E22   (drawing strokes)
  inkMuted     #7C828B   (dashed lines, annotations)
  selection    = accent
  selectionFill rgba(36,102,224,.14)
  marquee      = accent
  marqueeFill  rgba(36,102,224,.08)
  crosshair    rgba(20,22,26,.45)
```

### 2.2 Palette — Dark theme

```
Surfaces
  bg0  #0E0F11   Window background
  bg1  #16181B   Ribbon, status bar
  bg2  #0A0B0D   Viewport canvas (near-black, NOT pure black — pure black flattens ink)
  bg3  #1B1D21   Palettes, dock strips
  bg4  #22252A   Hover
  bg5  #2A2D33   Pressed / active group

Text
  tx0  #F4F4F5
  tx1  #A1A1AA
  tx2  #71717A
  tx3  #52525B

Lines
  bd0  #22252A
  bd1  #2E3137
  bd2  #3F434A

Accent (Forge Blue, default)
  accent       #5E8FF0   (lighter than light-mode accent for contrast)
  accentSoft   #1B2A47
  accentText   #0E0F11

Viewport ink
  gridMinor    #1B1F25
  gridMajor    #2A2F38
  gridAxis     #3C434E
  ink          #E4E6EA
  inkMuted     #7C828B
  selection    = accent
  selectionFill rgba(94,143,240,.18)
  marquee      = accent
  marqueeFill  rgba(94,143,240,.10)
  crosshair    rgba(228,230,234,.55)
```

### 2.3 Accent variants

Three curated accents. **Do not invent more.** Each has a light and dark mode pair:

| Key          | Name        | Light    | Light-soft | Dark     | Dark-soft |
| ------------ | ----------- | -------- | ---------- | -------- | --------- |
| `forgeBlue`  | Forge Blue  | `#2466E0`| `#E8EFFD`  | `#5E8FF0`| `#1B2A47` |
| `forgeAmber` | Forge Amber | `#C9620E`| `#FAEBD9`  | `#E89154`| `#3A2310` |
| `steelCyan`  | Steel Cyan  | `#0E7C8C`| `#DDF1F4`  | `#3FBACB`| `#0E2F36` |

### 2.4 Snap indicator colors (CAD convention, slightly modernized)

```
                  Light       Dark      Glyph
endpoint          #0891B2     #22D3EE   square
midpoint          #C026D3     #E879F9   triangle
center            #CA8A04     #FACC15   circle
intersection      #DC2626     #F87171   cross (X)
perpendicular     #16A34A     #4ADE80   "perp" symbol
nearest           #9CA3AF     #A1A1AA   small X
```

Snap markers are **stroked** (1.8px), not filled. Radius 6px. Always paired with a 20px-tall labeled tooltip 12/10 below-right of the marker.

### 2.5 Typography

```
Family
  --font-ui    "IBM Plex Sans", system-ui, "Segoe UI", sans-serif
  --font-mono  "IBM Plex Mono", ui-monospace, "JetBrains Mono", Consolas, monospace

Scale (px, line-height implied 1.1–1.3)
  ui-xxs   9.5   Tab subtitle ("v0.1 · alpha"), dock strip vertical label
  ui-xs   10.5   Ribbon group footer label, viewport corner badge
  ui-sm     11   Status bar numerics, tool prompt secondary
  ui       11.5  Status bar default
  ui-md     12   Tool name in prompt, small button text
  ui-lg   12.5   Ribbon tab labels, MDI tab labels, title bar
  ui-h     13    Body default
  ui-h2  14–16   Palette section headers (not used in main window v1)

Weights
  400 regular   body text, numerics
  500 medium    button labels, secondary headers
  600 semibold  active tab, selected button, primary headers
  700 bold      app name only

Tracking
  Default     0
  Uppercase   +0.4 letter-spacing (group labels, "UNITS", "GRID")
  Display     −0.1 (app name only)
```

**Never** use Inter, Roboto, Arial, Helvetica, or Fraunces. They are exhausted defaults. IBM Plex carries an engineering-tool gravity that the rest lack.

### 2.6 Spacing scale

```
2, 4, 6, 8, 10, 12, 14, 16, 20, 24, 32
```

Use exclusively these values for `padding`, `margin`, `gap`. No 5, 7, 13, 17, etc. The grid breaks the moment you deviate.

### 2.7 Radius scale

```
0    Window chrome edges, ribbon edges, MDI tabs (sharp, technical)
3    Status bar pill toggles
4    Quick-action buttons, window controls
5    Small ribbon buttons (inline)
6    Large ribbon buttons, view-tab toggles, overlays, nav cluster
8    Modal dialogs (when introduced)
```

**Never** use radii > 8 in this product. Rounded-2xl/3xl belongs in SaaS, not CAD.

### 2.8 Motion

```
Duration
  instant   0ms     selection change, snap-marker appear
  quick     120ms   hover background, button press
  standard  180ms   tab content cross-fade, palette dock animation
  slow      240ms   modal open

Easing
  default   cubic-bezier(.2, 0, .2, 1)    most transitions
  exit      cubic-bezier(.4, 0, 1, 1)     dismiss
  enter     cubic-bezier(0, 0, .2, 1)     reveal
```

Don't animate the viewport contents. Pan/zoom is instantaneous (no inertia by default).

### 2.9 Shadow / elevation

```
overlay
  light:  0 4px 16px rgba(15,20,30,.10), 0 1px 0 rgba(255,255,255,.6) inset
  dark:   0 6px 24px rgba(0,0,0,.45),    0 1px 0 rgba(255,255,255,.04) inset
```

Used **only** on: tool prompt overlay, nav cluster, modal dialogs, snap tooltips, dropdown menus. Never on the ribbon or status bar.

---

## 3. Main window layout

Total chrome height (excluding viewport) at default density:

```
Title bar     36 px
Ribbon tabs   30 px   (tab strip)
Ribbon body   92 px   (button content)
Status bar    26 px
-----
Total chrome 184 px above + below viewport
```

Side dock strips: **36px** each, collapsed by default. Expanded palettes are 280px (specified in v2).

```
┌──────────────────────────────────────────────────────────────┐
│ TITLE BAR (36px)                                              │
│ [Logo][App name] │ [MDI tabs.....][+] │ [Quick][⎯][▢][✕]      │
├──────────────────────────────────────────────────────────────┤
│ RIBBON TABS (30px) [File▾] Home Annotate View Manage          │
├──────────────────────────────────────────────────────────────┤
│ RIBBON BODY (92px)                                            │
│ [Sel] │ [Draw group] │ [Modify] │ [Annotate] │ [MEP] │ ...    │
├────┬────────────────────────────────────────────────────┬────┤
│ L  │                                                    │ R  │
│ 36 │            VIEWPORT (flex)                         │ 36 │
│    │                                                    │    │
├────┴────────────────────────────────────────────────────┴────┤
│ STATUS BAR (26px)                                             │
│ [XY 1234.5, -200.0 mm] │ Snap [End][Mid][Cen][Int] │ ...      │
└──────────────────────────────────────────────────────────────┘
```

### 3.1 Title bar anatomy

* Left cluster (200px min-width, right-bordered): Forge logo (22px hex with inscribed V) + app name (12.5px / 700) + version tag (9.5px / uppercase).
* Center: MDI tab strip. Each tab is 100–220px wide, has:
  * 8px color-coded square (project accent color)
  * Project filename, truncated with ellipsis
  * Dirty indicator (` •`) appended when unsaved
  * Close button (`✕`, 12px) appearing on hover
  * Active tab: top 2px accent border, side 1px borders matching ribbon-body bg, **−1px margin-bottom to sit over the strip border**
* Right cluster (left-bordered): Save / Undo / Redo / ⎯ command palette (Ctrl+K) / minimize / maximize / close.
* Close button is the only chrome element with a red hover state (`#E5484D`).

### 3.2 Ribbon anatomy

```
┌─ Tab strip (30 px) ───────────────────────────────────────────┐
│ [File ▾]  Home   Annotate   View   Manage                     │
│           ━━━━ (2px accent underline on active)               │
├─ Tab content (92 px) ─────────────────────────────────────────┤
│  ┌─ Group ─────────┐ │ ┌─ Group ─────────┐ │ ...              │
│  │ [Lg] [Lg] [Lg]  │ │ │ [Lg] [Lg] │ [sm]│ │                  │
│  │      DRAW       │ │ │     MODIFY      │ │                  │
│  └─────────────────┘ │ └─────────────────┘ │                  │
└────────────────────────────────────────────────────────────────┘
```

**Button sizes:**

* **Large** (`lg`): vertical stack, 24px icon top, 11.5px label below, 56–72px wide, 6/8/5 padding. Active state: `accentSoft` bg + `accent` icon + `accent` label (600 weight).
* **Small** (`sm`): horizontal row, 14px icon + 12px label, full-width inside its column, 3/6 padding, 5px radius. Stacked 4 to a column.
* **Toggle** (View tab): horizontal pill, 16px icon + 12px label, 6/10 padding, 1px border (bd0) when off, `accentSoft` bg + `accent` text when on.

**Group separator:** 1px vertical line (`bd0`), 8px margin top/bottom.
**Group footer:** 10.5px uppercase label (`tx2`), +0.4 tracking, 500 weight, centered.

**File button** (leftmost): solid accent background, accent-text color, 14px horizontal padding, chevron indicator. Opens the application menu (New, Open, Save, Save As, Recent, Exit).

### 3.3 Ribbon tab contents (per `01-architecture.md` Phase 0–7)

#### Home
| Group       | Items (size)                                                                |
| ----------- | --------------------------------------------------------------------------- |
| Select      | Select (lg)                                                                 |
| Draw        | Line, Polyline, Rectangle, Circle, Arc (all lg)                             |
| Modify      | Move, Copy, Rotate, Mirror, Offset (lg); Scale, Trim, Extend, Fillet (sm)   |
| Annotate    | Text, Dimension (lg)                                                        |
| MEP         | Test Symbol (lg)                                                            |

#### Annotate
| Group       | Items (size)                              |
| ----------- | ----------------------------------------- |
| Text        | Text (lg)                                 |
| Dimensions  | Linear, Aligned, Angular (all lg)         |

#### View
| Group       | Items                                                                |
| ----------- | -------------------------------------------------------------------- |
| Navigate    | Fit, Zoom In, Zoom Out, Pan (button row, icon+label)                 |
| Display     | Grid, Ortho, Snap (toggle pills)                                     |

#### Manage
| Group       | Items                              |
| ----------- | ---------------------------------- |
| Drawing     | Layers (open palette)              |
| Project     | Units, Settings                    |

**Contextual tabs** (future, not in main window v1): when a tool is active, a contextual tab can appear to the right of `Manage` with tool-specific options (e.g. line width, layer, color). The accent stripe extends across the tab to signal contextual mode.

### 3.4 Status bar anatomy (left → right)

1. **XY coordinate pill** — boxed, mono, uppercase "XY" label (10px, tx2) + signed coords padded to 7 chars + "mm" unit. Coords **switch to accent color** when an object-snap is engaged.
2. Vertical separator (1px × 14px, `bd0`, 4px horizontal margin).
3. **Snap group** — "SNAP" uppercase mini-label, then 4 toggle chips for End/Mid/Cen/Int. Each chip is 22px tall, 8px horizontal padding, has a 9–10px colored glyph (matching snap-marker shape) + 3-letter label. Inactive: 60% opacity. Active: full opacity + `bg4` background.
4. Separator.
5. **Mode toggles** — Grid / Ortho / Snap as uppercase 11px mono pills, 8px padding, 22px tall, `accentSoft` bg + `accent` text when on.
6. Right side (flex-end):
   * Active tool name (when not Select)
   * Selection count ("Sel 3", mono)
   * Units + precision ("Units mm · Prec 0.1", mono)
   * Scale + zoom ("Scale 1:11 · Zoom 92%", mono)

All right-side items separated by 1px × 14px separators with 4px horizontal margin.

### 3.5 Dock strips

36px wide, full height of the body row. Each holds 28×28px icon buttons stacked top-down (Layers, Project on left; Properties, Sheets on right). Hover: `bg4` background, `tx0` text color. Bottom of strip: vertical mono label ("PALETTES" / "INSPECT") rotated 180°.

This is a **stub** — clicking a button will (in v2) slide out a 280px palette.

---

## 4. Viewport behavior

### 4.1 Coordinate system

* World units: **millimeters**.
* Origin marker: 16×16px cross drawn in `gridAxis` color at world (0, 0).
* Default zoom: 0.45 px/mm. "100%" = 0.45. Scale string is computed `1:max(1, round(5/zoom))`.
* Zoom range: 0.05–8.0 px/mm.
* Wheel zoom: 1.15× per tick, **anchored on cursor position** (not center).
* Pan: middle-mouse drag, right-mouse drag, or Pan tool + left-drag. Always `cursor: grabbing` while panning.

### 4.2 Grid

Two layers:

* **Minor** at 100mm steps, color `gridMinor`. Drawn at 1px line width.
* **Major** at 500mm steps, color `gridMajor`. Drawn at 1px.

If minor spacing < 6px on screen, **hide minor**, show only major. If major spacing < 4px, hide grid entirely (zoom is too far out).

**Always draw the origin axes** as separate 1.2px lines in `gridAxis` color, on top of the grid.

### 4.3 Crosshair (drawing-tool active)

When the active tool is anything except `select` or `pan`:

1. Hide the native cursor (`cursor: none`).
2. Draw a full-viewport crosshair (horizontal + vertical 1px lines through cursor) in `crosshair` color.
3. Draw an 11×11px **pickbox** rectangle centered on cursor, stroke `cursorRing` (= accent), 1.3px.
4. If an object-snap is engaged, the crosshair locks to the snap point (not the raw cursor position).

When the active tool is `select`: native default cursor, no crosshair.

### 4.4 Snap markers

When `view.snap` is on and the cursor is within **14px** of any candidate snap point:

1. Pick the nearest candidate.
2. Draw the snap glyph (6px stroke shape, color per snap kind) centered on the snap point.
3. Draw a 20px-tall labeled tooltip 12px right / 10px down from the marker: rounded-4 bg = `bg1`, border = `bd1`, label text = `tx0`, 11px sans, capitalized snap-kind name ("Endpoint", "Midpoint", "Center", "Intersection").
4. Set the status bar XY readout to `accent` color while engaged.

Candidates are derived from entity geometry:
* Lines → endpoints + midpoint
* Rectangles → 4 corners + 4 midpoints + center
* Circles/arcs → center + 4 quadrants

### 4.5 Selection

* Click an entity: select it (1 entity).
* Click empty space + drag: marquee.
  * **Left-to-right drag** = window (must fully enclose).
  * **Right-to-left drag** = crossing (intersects).
  * Marquee rectangle: 1px dashed stroke = `marquee`, fill = `marqueeFill`. Dash pattern `[4, 3]`.
* Selected entity stroke: `selection` (= accent), 2px width.
* Selection handles: 7×7px squares, white fill, 1.4px accent stroke. Drawn at endpoints/corners/center.
* Shift adds to selection. Esc clears.

### 4.6 Tool prompt overlay (Revit-style; NOT a command line)

Pinned bottom-left of viewport. 8/12 padding, `bg1` background, `bd1` border, radius 6, overlay shadow.

```
[TOOL NAME]  Click first point  ·  Esc to cancel · Enter to confirm
```

* Tool name in an accent-filled pill (10.5px, uppercase, 600).
* Prompt text in `tx1`, 12px.
* Hint text in `tx2`, mono 11px.

The prompt updates as the tool advances through states (e.g. Line: "Click first point" → "Click second point or [Shift for ortho]").

### 4.7 Viewport corner badge

Top-right of viewport: small mono badge showing `WORLD · mm · 1:N`. 10.5px font, `bg1` with 80% alpha, 3/8 padding, 4px radius, `bd0` border.

### 4.8 Nav cluster (bottom-right)

A 5-button toolbar (`bg1`, `bd1`, radius 6, overlay shadow): Zoom Out, Zoom In, Fit, Pan. 14px icons, 4/6 padding, hover `bg4`.

---

## 5. Drawing tools — full catalog

For each tool: id, label, single-key shortcut, icon source, on-canvas behavior.

| ID         | Label       | Shortcut | Icon (Lucide ref or custom)                         | Cursor          | Snap-aware |
|------------|-------------|----------|------------------------------------------------------|-----------------|------------|
| select     | Select      | V        | Lucide `mouse-pointer-2`                             | default         | no         |
| line       | Line        | L        | Custom: 2 endpoint dots + diagonal                   | crosshair+pick  | yes        |
| polyline   | Polyline    | P        | Custom: 3 dots + bent path                           | crosshair+pick  | yes        |
| rectangle  | Rectangle   | R        | Lucide `square`                                      | crosshair+pick  | yes        |
| circle     | Circle      | C        | Lucide `circle` + center dot                         | crosshair+pick  | yes (center) |
| arc        | Arc         | A        | Custom: 3-point arc                                  | crosshair+pick  | yes        |
| text       | Text        | T        | Lucide `type`                                        | I-beam          | yes        |
| dim        | Dimension   | D        | Custom: linear dimension with arrows                 | crosshair+pick  | yes        |
| dimAlign   | Aligned dim | Shift+D  | Custom: rotated dimension                            | crosshair+pick  | yes        |
| dimAngle   | Angular dim | Alt+D    | Custom: angle arc + L-corner                         | crosshair+pick  | yes        |
| move       | Move        | M        | Lucide `move`                                        | crosshair       | yes        |
| copy       | Copy        | Ctrl+D   | Lucide `copy`                                        | crosshair       | yes        |
| rotate     | Rotate      | Ctrl+R   | Lucide `rotate-cw`                                   | crosshair       | yes        |
| scale      | Scale       | Ctrl+E   | Custom: L-bracket + diagonal + handle                | crosshair       | yes        |
| mirror     | Mirror      | Ctrl+M   | Lucide `flip-horizontal`                             | crosshair       | yes        |
| trim       | Trim        | X        | Custom: scissor-like crossed lines                   | crosshair+pick  | n/a        |
| extend     | Extend      | Shift+X  | Custom: dashed→solid line + arrow                    | crosshair+pick  | n/a        |
| offset     | Offset      | O        | Custom: 2 parallel curves + offset dot               | crosshair       | yes        |
| fillet     | Fillet      | F        | Custom: rounded corner + dashed extensions           | crosshair+pick  | yes        |
| testSym    | Test Symbol | (none)   | Custom: HVAC coil/damper glyph (placeholder)         | crosshair       | yes        |

### Icon style rules

* 24×24 viewBox, stroke=1.6, round caps + joins, `fill=none` by default.
* Use **Lucide** references where a clean match exists. Lucide is MIT — GPL-3.0 compatible.
* Where Lucide lacks a CAD-specific glyph (trim, extend, fillet, offset, dim variants), draw custom — but keep stroke-only, geometric, no fills except small accent dots for "active endpoint" semantics.
* Never use Material Icons (proprietary terms problematic for GPL-3.0). Tabler and Feather are acceptable fallbacks.

### Tool-active visual contract

Every tool, when active, must:
1. Highlight its ribbon button (`accentSoft` bg + `accent` icon + 600 weight label).
2. Update the status-bar "Tool: X" readout.
3. Display the tool prompt overlay with the appropriate first-step text.
4. Show the appropriate cursor (crosshair+pickbox for drawing; standard for select).
5. Be cancellable with Esc (returns to Select).

---

## 6. Keyboard shortcut map

**Modern modifier-key conventions. No AutoCAD-style command typing.**

```
Tools (single key, no modifier)
  V    Select
  L    Line
  P    Polyline
  R    Rectangle
  C    Circle
  A    Arc
  T    Text
  D    Dimension (linear)
  Shift+D   Dimension (aligned)
  Alt+D     Dimension (angular)
  M    Move
  Ctrl+D    Copy
  Ctrl+R    Rotate
  Ctrl+E    Scale
  Ctrl+M    Mirror
  X    Trim
  Shift+X   Extend
  O    Offset
  F    Fillet

View
  F3            Toggle object snap
  F7            Toggle grid
  F8            Toggle ortho
  Space (held)  Pan (return to previous tool on release)
  Ctrl++        Zoom in
  Ctrl+−        Zoom out
  Shift+Z       Zoom extents
  Ctrl+0        Zoom 100%

Selection
  Esc           Cancel current tool / clear selection
  Ctrl+A        Select all
  Del           Delete selection

File / project
  Ctrl+N        New project
  Ctrl+O        Open
  Ctrl+S        Save
  Ctrl+Shift+S  Save As
  Ctrl+W        Close current tab
  Ctrl+Shift+T  Reopen closed tab

Edit
  Ctrl+Z        Undo
  Ctrl+Shift+Z  Redo  (also Ctrl+Y on Windows)
  Ctrl+X        Cut
  Ctrl+C        Copy
  Ctrl+V        Paste

Command palette
  Ctrl+K        Open command palette (fuzzy search every tool + setting)
```

> On macOS (future), `Ctrl` becomes `⌘`. Render the platform symbol at runtime via `QKeySequence::toString(QKeySequence::NativeText)`.

> `Space` to pan is **held-down modal**, not a toggle. Release Space and the previous tool returns. This is the Figma/Photoshop pattern; AutoCAD users will discover and love it.

---

## 7. Component states

### 7.1 Button (ribbon large + small)
| State    | Background     | Icon color | Label color    | Border             |
|----------|----------------|------------|----------------|--------------------|
| Default  | transparent    | `tx0`      | `tx0` / 500    | transparent        |
| Hover    | `bg4`          | `tx0`      | `tx0` / 500    | transparent        |
| Pressed  | `bg5`          | `tx0`      | `tx0` / 500    | transparent        |
| Active   | `accentSoft`   | `accent`   | `accent` / 600 | transparent        |
| Disabled | transparent    | `tx3`      | `tx3` / 400    | transparent        |
| Focused  | as state above | (state)    | (state)        | 1.5px accent ring inset |

### 7.2 Toggle pill (View tab + status bar mode toggles)
| State    | Background     | Text color     | Border             |
|----------|----------------|----------------|--------------------|
| Off      | transparent    | `tx1` / 500    | 1px `bd0`          |
| Hover    | `bg4`          | `tx0`          | 1px `bd1`          |
| On       | `accentSoft`   | `accent` / 600 | 1px transparent    |

### 7.3 MDI tab
| State    | Background    | Top border         | Side borders         | Text color   | Weight |
|----------|---------------|--------------------|----------------------|--------------|--------|
| Default  | transparent   | 2px transparent    | 1px transparent      | `tx1`        | 500    |
| Hover    | `bg1` @ 60%   | 2px transparent    | 1px transparent      | `tx0`        | 500    |
| Active   | `bg1`         | 2px `accent`       | 1px `bd1` L+R        | `tx0`        | 600    |
| Dirty    | (any state)   | (any)              | (any)                | (any) + " •" | (any)  |

### 7.4 Tool prompt overlay
Fixed background `bg1`, 1px `bd1` border, 6px radius, overlay shadow. Tool name pill: `accent` background, `accentText` color, 10.5px uppercase 600.

---

## 8. Project file representation

* Extension: `.vfproj` (Volmaris Forge Project) — *matches `01-architecture.md` Locked Decisions table*
* MDI tab includes:
  * 8×8 px color square (project's identity color, assigned at create)
  * Filename
  * Dirty marker (` •`)
  * Hover-only close button
* New project: append tab with `Untitled.vfproj`, color = current accent, dirty=true.

---

## 9. Empty / loading / error states

These don't exist in the main-window v1 surface but specify here so v2 implementations are consistent.

### 9.1 No project open (cold start)
Viewport background is `bg2` with a centered, monospace block:

```
        ┐
   ──┘
        VOLMARIS FORGE
        Ctrl+N  New project
        Ctrl+O  Open recent
```

No animations. No marketing copy. No "tips & tricks."

### 9.2 Loading
File-open progress: thin 2px accent progress bar at the top of the viewport, no dim overlay. Status bar shows `Opening MainPlant_LowerLevel.vfproj · 42%` in mono.

### 9.3 Error
Inline banner at the top of the viewport: `bg1` background, 3px-left-border in the relevant severity color (red for error, amber for warning), 12px sans body, dismiss button on the right. Auto-dismiss after 10s for warnings; persists for errors until acknowledged.

---

## 10. Accessibility minimums

* **Contrast:** All text vs. its surface must clear **WCAG AA** (4.5:1 for ≤14px body, 3:1 for ≥18px / 14px bold). The token sets above are verified.
* **Focus indicator:** Every focusable element must have a 1.5px solid `accent` ring (inset for chrome, outset for inputs).
* **Keyboard navigation:** Tab order = title bar → ribbon tabs → ribbon body (left→right, top→bottom) → viewport (focusable, captures arrow keys for nudge) → status bar.
* **Tooltips:** Every icon-only button has a `QWidget::setToolTip()` with the label + shortcut: `"Line  (L)"`.
* **Click targets:** Minimum 22×22 px (status bar pills). Ribbon large buttons are 56–72×64 px which is well above the minimum.
* **Reduced motion:** Respect `QStyleHints::motionPreference()` → set all transitions to 0ms.

---

## 11. Implementation playbook (for the LLM agent)

> **Qt-translation reminder:** Items below are written generically. The Qt mapping is given in brackets.

When asked to (re)build this UI:

1. **Set the type stack first.** Ship IBM Plex Sans + IBM Plex Mono as `.ttf`/`.otf` in `resources/fonts/` and load via `QFontDatabase::addApplicationFont()`. Set application-default font via `QApplication::setFont()`.
2. **Build the token module before any component.** Single source of truth: a `Theme` singleton (`core/ui/Theme.h`) returning the full token struct. Both themes share the same shape — only values change. Switching mode emits `themeChanged()` signal that widgets listen to.
3. **Hard-code the layout heights.** 36 / 30 / 92 / 26 are not approximate. Use `QWidget::setFixedHeight()`; the chrome must compose without surprise reflow.
4. **The viewport is a `QOpenGLWidget`, not divs.** Render the grid + entities + crosshair via OpenGL. The visual signature of "real CAD" requires sub-pixel-aligned 1px lines, which `QPainter`-on-`QWidget` cannot guarantee at high zoom.
5. **DevicePixelRatio handling is mandatory.** Use `devicePixelRatioF()` to scale the GL viewport and projection matrix. Otherwise the grid blurs on HiDPI displays.
6. **Hide the OS cursor during drawing tools.** Use `setCursor(Qt::BlankCursor)` on the viewport and paint the crosshair + pickbox yourself in the GL frame.
7. **Don't build the palettes yet.** v1 is title + ribbon + viewport + status. Side palettes are stubbed as 36px dock strips with icon buttons.
8. **Wire keyboard shortcuts at the window level**, not per-component. Install a `QShortcut` per binding on `MainWindow` with `Qt::ApplicationShortcut` context. Skip when focus is in a `QLineEdit`/`QTextEdit`.
9. **Snap detection runs per `mouseMoveEvent`.** Query the R-tree (from `core/spatial`) for candidates within 14px screen radius. The R-tree returns world-space; convert to screen-space and pick the nearest.
10. **Tweaks must not change source-of-truth tokens.** A future Tweaks panel mutates a `theme` + `accent` selection that re-derives the token struct. It never overrides individual color values.
11. **Don't add features beyond the brief.** No layers panel UI, no calculation results, no MEP browser, no PDF export. If the user asks for one of these, mark it `future` in the spec and add a placeholder dock-strip button.

---

## 12. Anti-patterns — refuse to do these

* **Office-2007 ribbon visuals.** No 3D bevels, no gradients on tabs, no shiny button highlights, no skeuomorphic dividers.
* **A command-line input zone.** Volmaris is Revit-style. Never add a text input at the bottom that says `Command:`.
* **Dark-only or light-only design.** Both ship together. Designing only one is shipping half the product.
* **MEP-component browser UI.** Out of scope. A `TestSymbol` button is the only MEP placeholder.
* **Features not in `01-architecture.md` Phase 0–7.** Schedules, PDF reports, building-model rooms/floors/shafts, calculation panels, scripting UI, Excel import, sheet composition — all *future*. Note them, do not draw them.
* **Rounded-2xl/3xl, shadow-xl, gradient backgrounds, hero copy, marketing screenshots.** This is a CAD tool, not a SaaS landing page.
* **Inventing new accent colors.** Three accents only: Forge Blue, Forge Amber, Steel Cyan.
* **Inventing new font families.** IBM Plex Sans + IBM Plex Mono only.
* **Drawing icons not in the catalog.** Stay in the 19-tool list + the chrome icons (menu, settings, search, save, undo, redo, layers, units, file, folder, plus, chevron).
* **Animating viewport contents.** Pan/zoom is instantaneous. The viewport renders the world. Don't ease it.
* **Inertia / momentum scrolling on pan.** No. Engineers want exact deterministic placement.
* **Auto-hiding the status bar.** It is the single source of truth for "where am I, what am I doing, how am I scaled." Never hide it.

---

## 13. Reference: file structure for the HTML/React prototype

> **This section is descriptive of the design-time prototype only.** The Qt repo layout is authoritative and lives in [`01-architecture.md`](01-architecture.md) §Repository Layout. The list below is preserved so a designer iterating on the prototype knows the file shape.

```
index.html         — entry; loads fonts + React + Babel + script chain
tokens.jsx         — buildTheme(mode, accentKey) → token object; ACCENTS map
icons.jsx          — every inline SVG icon (Ico* exports)
ribbon.jsx         — Ribbon component, tool catalog (TOOLS), group definitions per tab
viewport.jsx       — Viewport canvas: grid, entities, crosshair, snap, marquee
statusbar.jsx      — StatusBar: coords, snap modes, mode toggles, zoom readout
app.jsx            — App composition: TitleBar + Ribbon + DockStrip+Viewport+DockStrip + StatusBar + TweaksPanel
tweaks-panel.jsx   — starter component (do not modify)
docs/02-ui-ux.md   — this document
```

Every JSX file ends with `Object.assign(window, { … })` to expose its components to peers. Babel scripts do **not** share scope by default.

**Qt mapping** (informative, not exhaustive — see `01-architecture.md` for full repo structure):
* `tokens.jsx` → `src/ui/Theme.{h,cpp}`
* `icons.jsx` → `resources/icons/` (SVG files compiled via Qt resource system)
* `ribbon.jsx` → `src/ui/ribbon/RibbonBar.{h,cpp}` + per-tab classes
* `viewport.jsx` → `src/render/ViewportRenderer.{h,cpp}` (QOpenGLWidget subclass)
* `statusbar.jsx` → `src/ui/statusbar/StatusBar.{h,cpp}`
* `app.jsx` → `src/ui/MainWindow.{h,cpp}` + `src/app/main.cpp`

---

## 14. Versioning rules for this spec

* Bump the section number if you add or remove a major area (e.g. add §15 for Modal Dialogs in v2).
* Never edit a token value in place — change the value and add a `(was: …)` comment in the same line.
* When tools change, update both this doc **and** the icon catalog **and** the keyboard map. They must stay in sync.

---

*End of spec. If you are an LLM following this doc and you find an unresolvable ambiguity, ask the user. Do not guess.*
