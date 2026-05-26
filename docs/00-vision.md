# Volmaris Forge — Product Vision

> **Living document.** This is intentionally evolving — not all of it gets built. It captures the *north star*. The current build target is `01-architecture.md`. When an item here matures into something we want to build, it graduates into the architecture doc.
>
> If you (the user) had a clearer picture, we'd freeze it. You don't yet. That's fine. Add to this as ideas crystallize; nothing here is a commitment.

---

## Why This Tool Exists

HVAC schematic drawing today is broken across three flavors of tools, and Volmaris Forge exists because none of them do the job well:

| Tool flavor | What's wrong |
|---|---|
| AutoCAD-style (LibreCAD, BricsCAD) | A schematic is just lines on paper. No logic, no system awareness, no calculations. Blocks barely help. |
| Calculation-program-style (Dendrit, similar) | Strong calcs but rigid, limited drawing flexibility, can't express anything outside the calc-engine's model. |
| BIM-attached schematics (Revit's schematic view) | Tied to a full 3D model that doesn't exist yet during early design. Heavy. Slow. Schema work happens after pipe layout, not before. |

**Concrete pain points** the program must address:

1. **Schemas are usually drawn *after* the layout** — once the project is complex, late. We want schemas usable as the **early design tool**, before pipes/ducts are laid out.
2. **Editing entire floors is brutal** — shifting a floor's elevation means manually selecting everything on that level and moving it. Should be one operation.
3. **Inserting a room into an existing row** of rooms (e.g., adding a new apartment between two existing ones) is painful. Should be insert-and-shift.
4. **Data flows are visual only** — moving data from a model to the schema means eyeball-matching, or matching Excel sheets by room name/number. Should be programmatic.
5. **UI/UX in existing tools is decades old.** This alone is a real moat — even a tool that only matched feature parity but with a *modern* UX would have a market.

---

## Inspirations & Anti-Inspirations

**Liked:**
- **CypeHVAC** — its ability to generate **PDF reports for any selected portion of a system line**. Reference example: water distribution pipes "consult checks" report. The format is repeatable per check:
  - **Header bar**: check name (e.g., "Check: Maximum velocity")
  - **Summary box** (right-aligned): limit value, actual computed value, pass/fail icon (✓ / ✗)
  - **Formula**: the equation in proper mathematical notation
  - **Where clause**: each symbol expanded with its name + current resolved value + unit (e.g., "V — Velocity = 1.05 m/s", "Qw — Flow = 1.38 l/s", "D — Internal diameter = 40.9 mm")
  - Multiple such check-blocks stack vertically on the same page; the report is per pipe / per segment / per system as the user selects.
  - Reference image: place at `docs/images/cype-report-example.png` when uploaded.
- **Revit's pipe behavior** — pipes maintain connections when moved; you can select at multiple levels of connectivity (segment, run, system); fittings auto-insert at intersections.
- **Revit's schedule (table) system** — tables that are *live views* into the model, edit either side and both update.
- **Dynamo (Revit)** — user-writable visual/scripted logic that runs against the model.

**Disliked:**
- AutoCAD's command-line-only mental model.
- Revit's monolithic, slow, all-or-nothing project file.
- Existing schematic tools' visual styling — looks like 1995.

---

## The Big Themes (organized by feature bucket)

### 1. Drawing Experience
- **Revit-like connection behavior.** Pipes/ducts maintain connectivity when endpoints move. Drag one component and connected runs follow or stretch as appropriate.
- **Hierarchical selection.** Click once = segment. Click again = full run. Click again = full system. Or modifier keys to choose level.
- **Quick edit modes** — drag a floor band to shift it; insert-and-shift rooms in a row; bulk relayout by reflowing.
- **Modern feel** — smooth, no surprises, no command-line walls of text. Properties palette is the source of truth.

### 2. Intelligence — "the program knows what it's drawing"
- **System awareness** — every element belongs to a system; the program knows what flows through it, what it connects to, what's compatible.
- **Building model awareness** — the schema knows about rooms, floors, shafts, staircases, technical rooms. Not full 3D BIM — just the topology that schemas care about.
- **Validation** — connectors with incompatible system types or sizes flag warnings. Open connectors highlighted. Loops/orphans detected.
- **Auto-routing hints** — given two components on different floors with a shaft between them, the program can suggest/auto-draw the connecting segments.

### 3. Calculations
- **Graph-based propagation** — schemas are graphs of connected components; the program walks them to compute flow, power, pressure drop, etc.
- **Aggregations along a line** — sum flow downstream of any point; power per branch; total per system.
- **User-definable formulas** — the user adds custom calculations via expressions or scripts.
- **Python scripting (Dynamo-like)** — let users write services in Python that run against the model. Major lift; capture as an open question (see below).
- **Pre-defined calculation modules** — heat-load, flow-balance, pump sizing, etc., as optional add-ons.

### 4. Early Design Mode
- **Fast iteration.** Define a few shafts and levels, drop in a few rooms, mark a technical room location — the program proposes a basic schematic layout. User refines.
- **Bulk parameter input.** Select all rooms on a floor, set W/m² (or W total) — power load propagates through the system.
- **What-if cheaply.** Change a single parameter, see all downstream numbers update.

### 5. Data Interop
- **Import Excel of rooms + numbers + data** → program creates rooms on levels automatically.
- **External-object mapping.** Map an imported model's objects to internal Volmaris components (e.g., "this Revit family = this Volmaris valve type").
- **Export to common formats.** PDF (per-line reports, like CypeHVAC), CSV/Excel, possibly IFC for BIM round-tripping.

### 6. Reporting & Sheets
- **Per-system PDF reports** — select any line/branch, generate a structured PDF (data sheet, components, calculated values). CypeHVAC-style.
- **Sheet composition.** Drop views, tables, title blocks onto sheets for printing/PDF output.

### 7. Tables (Schedules)
- **Live tables** — Revit-schedule-like. A table is a query view into the model; editing either side updates the other.
- **Imported tables** — also support static imported tables for layout reference.

---

## Constraints / Principles

- **Open source under GPL-3.0.** The engineering community deserves a fundamentally better tool, owned by the community.
- **Widely applicable to engineers across the board.** Even within HVAC, ensure no Europe-only or USA-only assumptions creep in. Discipline scope (HVAC-only vs full MEP vs broader) is an open question — see below.
- **Composition not inheritance** (see architecture doc). MEP elements own/generate CAD primitives; they don't *are* CAD primitives.
- **No premature scope expansion.** This vision doc is the long-game spec. The *build target* is whatever is in `01-architecture.md` today.

---

## Open Questions (to be resolved as the vision matures)

| Question | Why it matters | Status |
|---|---|---|
| **Discipline scope:** HVAC only, all MEP, or include process piping / P&ID? | Affects abstractions in the system/component model. Easier to build narrow now, generalize later. | Open |
| **Scripting language:** Python (Dynamo-like, via embedded CPython + pybind11), JavaScript (QJSEngine, lightweight), or visual node-based (like Dynamo proper)? | Each has very different licensing, distribution, and complexity costs. | Open |
| **Building model depth:** lightweight topology only (rooms, floors, shafts), or extend toward IFC-compatible BIM-lite? | If IFC import/export ever matters, the building model needs IFC-compatible structure from the start. | Open |
| **Calculation engine model:** in-process synchronous, or async/incremental like a spreadsheet engine? | Affects responsiveness on large projects. | Open |
| **Selection-hierarchy UX:** modifier-key cycling (Ctrl+click), repeated-click cycling (Revit-style), or selection-context-tabs in the ribbon? | Drives a core daily interaction. Should be prototyped before committing. | Open |
| **Sheet / view system:** Revit-style (named views, sheets, viewports), AutoCAD-style (paper space tabs), or something hybrid? | Big UX decision for the eventual print/PDF workflow. | Open |
| **PDF report engine:** custom Qt-based templating, Markdown + Pandoc, HTML + headless Chrome, or a real reporting library (LimeReport, KReport)? | Affects template authoring UX and output quality. | Open |
| **Auto-layout intelligence:** rule-based heuristics or ML-assisted? | Rule-based is realistic now; ML is a research project. | Open |

---

## Things Explicitly NOT in the Vision (yet)

- Real-time multi-user collaboration. (Maybe someday.)
- Mobile / tablet versions.
- Web client. (Native desktop is the focus.)
- Full 3D modeling. (Volmaris Forge is a schematic tool; 3D BIM stays in other tools.)
- AI-driven generative design. (Open question, but currently not in scope.)

---

## How to Use This Doc

- **Adding ideas:** drop them under the appropriate theme. Don't over-formalize — bullet points are fine. We'll restructure when patterns emerge.
- **Promoting ideas to "build now":** when an item is concrete enough to spec, move it into `01-architecture.md` with locked decisions, acceptance criteria, and impl notes. Then either delete it here or mark it "→ now in architecture doc".
- **Resolving open questions:** when one of the questions above is decided, move the decision to `01-architecture.md` and strike it through here with the resolution.

This doc is intentionally *evolving*, not authoritative. The architecture doc is authoritative for what's getting built.
