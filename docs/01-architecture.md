# Volmaris Forge — v1 Implementation Architecture

> **Authoritative build contract.** This is what the implementation agent builds. Long-term direction and "maybe-someday" features live in [`00-vision.md`](00-vision.md) — do not pull from there unless an item has been explicitly promoted into this document.

---

## Context

**Volmaris Forge** is a brand-new 2D HVAC schematic editor, built from the ground up rather than on top of OpenCADStudio. The product philosophy: a high-performance CAD drawing engine first, with a MEP semantic layer composed on top (CAD entities = geometry; MEP elements = logic/data/connectors that *generate* CAD primitives but remain the real selectable object).

v1 focuses on **landing the core CAD drawing experience first** — primitives, snapping, layers, project I/O, ribbon UI, MDI shell — so the drawing UX feels excellent before MEP semantics get layered in. The architecture for MEP and the Revit-like Component/Family system must be scaffolded in v1 (interfaces + one working example) so future work just fills in stubs, not refactors.

The working directory `C:\Users\lic\Desktop\3 SWD\Schema\Drawing` is empty — clean greenfield. Prior prototypes elsewhere are deliberately ignored.

---

## Locked Decisions

| Area | Decision |
|---|---|
| Language / framework | **C++20 + Qt 6 LTS (LGPL)** |
| Renderer | **OpenGL via `QOpenGLWidget`** (custom render loop with batched draw calls) |
| Build / deps | **CMake + vcpkg**; Boost.Geometry (R-tree), SQLite, GTest |
| Target OS | **Windows + Linux** (architected cross-platform; macOS deferred) |
| App shell | **MDI** — multiple projects open in one window |
| Project file | **SQLite single-file** — extension `.vfproj` |
| Library file | **SQLite single-file** — extension `.vfclib` (component libraries) |
| Units | **Millimeters internal (double precision)**; configurable display (mm/cm/m/inch/ft) |
| Layers | **Simple**: name, color, visibility, lock (no linetype/lineweight in v1) |
| Snap modes (v1) | **Basic**: grid + endpoint/midpoint/center + ortho |
| Drawing tools (v1) | **Standard CAD**: line, polyline, rectangle, circle, arc, text, dimensions (linear/aligned/angular), modify (move/copy/rotate/scale/mirror/trim/extend/offset/fillet) |
| UI style | **Revit-style ribbon (custom-built on Qt widgets)** + properties palette + contextual tabs |
| Undo/redo | **Command pattern** |
| Spatial index | **R-tree (Boost.Geometry)** |
| Perf target | **100k entities @ 60fps pan/zoom; <2s project load** |
| Testing | **GTest unit tests + golden-image render tests**, CI on Windows + Linux |
| **License** | **GPL-3.0** (open source) |
| **Repo** | **Public GitHub from day 1** with GitHub Actions CI |
| MEP/Component scaffold | **Interfaces + one minimal working example** (a placeholder `TestSymbol` you can place and select to prove the layering works end-to-end) |

**License-compatibility note for the impl agent:** Qt 6 used under LGPL is compatible with GPL-3.0 via dynamic linking. SQLite is public domain. Boost is BSL-1.0. GTest is BSD-3. All chosen dependencies are GPL-3.0-compatible.

---

## Architecture — Layered Model

```
Document (.vfproj, MDI-managed)
 ├── CAD Model           ← fully implemented in v1
 │    ├── Layers
 │    ├── Entities: Line, Polyline, Rectangle, Circle, Arc, Text, Dimension, Hatch, Block
 │    └── EntityStore (id-keyed) + R-tree spatial index
 │
 ├── MEP Model           ← interfaces + 1 example in v1
 │    ├── IMepElement, IConnector, ISystemType (interfaces)
 │    ├── Systems registry (stub)
 │    └── TestSymbol (concrete example: places, selects, owns a generated CadPolyline)
 │
 ├── Component System    ← interfaces + 1 example in v1
 │    ├── IComponentDefinition, IComponentType, IComponentInstance
 │    ├── Parameter bindings, label bindings (interface only)
 │    └── In-memory definition for TestSymbol
 │
 └── Render Layer        ← fully implemented in v1
      ├── RenderPrimitive (line, polyline, text, etc. — GPU-friendly form)
      ├── RenderCache (per-entity cached primitives, dirty-tracked)
      ├── BatchBuilder (groups primitives for minimal draw calls)
      └── ViewportRenderer (QOpenGLWidget with pan/zoom/cull)
```

**Composition principle:** A MEP element *owns* generated CAD primitives via `OwnerElementId`. Clicking a generated polyline selects the owning MEP element, not the raw line. Manual CAD entities have no `OwnerElementId` and behave as normal drafting objects.

---

## Repository Layout

```
volmaris-forge/
├── CMakeLists.txt                  (top-level)
├── vcpkg.json                      (manifest: qtbase, boost-geometry, sqlite3, gtest)
├── .github/workflows/ci.yml        (Win + Linux build + test)
├── README.md                       (project intro, build instructions, contribution guide)
├── LICENSE                         (GPL-3.0 full text)
├── CONTRIBUTING.md                 (how to contribute)
├── docs/
│   ├── 00-vision.md                (long-term product vision — living doc)
│   ├── 01-architecture.md          (this doc — authoritative build contract)
│   └── 02-ui-ux.md                 (UI/UX spec — to be written next)
├── src/
│   ├── core/                       (no Qt deps — pure C++)
│   │   ├── geometry/               (Vec2, Bbox, line/arc math)
│   │   ├── entity/                 (Entity, EntityId, EntityStore)
│   │   ├── spatial/                (RTreeIndex wrapping Boost.Geometry)
│   │   ├── command/                (ICommand, CommandStack, undo/redo)
│   │   └── document/               (Document, Layer)
│   ├── cad/                        (CAD entity types)
│   │   ├── entities/               (CadLine, CadPolyline, CadCircle, CadArc, CadText, CadDimension, CadBlock)
│   │   └── tools/                  (LineTool, PolylineTool, ... — ITool interface)
│   ├── mep/                        (scaffold + 1 example)
│   │   ├── interfaces/             (IMepElement, IConnector, ISystemType)
│   │   ├── component/              (IComponentDefinition, IComponentType, IComponentInstance, ParameterBinding)
│   │   └── test_symbol/            (TestSymbol — proves end-to-end)
│   ├── render/                     (Qt + OpenGL)
│   │   ├── RenderPrimitive.{h,cpp}
│   │   ├── RenderCache.{h,cpp}
│   │   ├── BatchBuilder.{h,cpp}
│   │   ├── ViewportRenderer.{h,cpp} (QOpenGLWidget subclass)
│   │   └── shaders/                (.vert / .frag for line/polyline/text)
│   ├── io/                         (persistence)
│   │   ├── ProjectStore.{h,cpp}    (SQLite schema, save/load .vfproj)
│   │   └── LibraryStore.{h,cpp}    (.vfclib — stub in v1)
│   ├── snap/                       (snap engine + indicators)
│   ├── ui/                         (Qt widgets)
│   │   ├── MainWindow.{h,cpp}      (MDI host)
│   │   ├── DocumentWindow.{h,cpp}  (one project = one MDI sub-window)
│   │   ├── ribbon/                 (custom ribbon: RibbonBar, RibbonTab, RibbonPanel, RibbonButton)
│   │   ├── palettes/               (PropertiesPalette, LayersPalette)
│   │   └── statusbar/              (coord readout, snap toggles, unit display)
│   └── app/
│       └── main.cpp
└── tests/
    ├── unit/                       (GTest: geometry, entity store, commands, snap, R-tree)
    └── render/                     (golden-image tests — diff against PNG baselines)
```

**Key boundary:** `src/core/` has **zero Qt dependencies** — pure C++, unit-testable in isolation. Qt only enters at `render/`, `ui/`, and `app/`. This keeps the engine portable and the tests fast.

---

## Module-Level Detail

### `core/entity` — EntityStore
- `EntityId` — `uint64_t` strong typedef.
- `EntityStore` — owns all entities by `EntityId`. Hash map for O(1) lookup. Emits dirty events on mutate.
- Every entity carries: `id`, `layerId`, optional `ownerElementId` (for MEP-generated CAD), `bbox()` method.
- Document holds one `EntityStore`. R-tree is kept in sync via dirty events.

### `core/command` — Command pattern
- `ICommand { virtual void apply(Document&); virtual void revert(Document&); virtual std::string label(); }`
- `CommandStack` — undo/redo with merging for fast repeated commands (e.g., drag).
- Tools build commands and push them; tools never mutate the document directly.

### `core/spatial` — R-tree
- Thin wrapper around `boost::geometry::index::rtree`. Inserts `(bbox, entityId)` pairs.
- Methods: `query(bbox)`, `nearest(point, k)`, `insert/remove/update(entityId, bbox)`.

### `render/` — OpenGL pipeline
- `RenderPrimitive` — line strips, triangles for filled shapes, glyph quads for text.
- `RenderCache` — per-entity cached primitives. Invalidated on entity mutate (driven by EntityStore dirty events).
- `BatchBuilder` — groups primitives by shader/state; emits minimal draw calls.
- `ViewportRenderer` (QOpenGLWidget) — handles pan/zoom matrices, view-frustum culling via R-tree, dirty-region redraw when zoom unchanged.
- Shaders: one for lines (with width via geometry shader / triangle expansion), one for filled, one for text (signed-distance-field glyph atlas, generated at startup from a chosen font).

### `io/ProjectStore` — SQLite schema (sketch)
```sql
CREATE TABLE meta(key TEXT PRIMARY KEY, value TEXT);          -- version, units, etc.
CREATE TABLE layers(id INTEGER PK, name TEXT, color INT, visible INT, locked INT);
CREATE TABLE entities(id INTEGER PK, type TEXT, layer_id INT, owner_id INT NULLABLE, data BLOB);
CREATE INDEX idx_entities_layer ON entities(layer_id);
CREATE INDEX idx_entities_owner ON entities(owner_id);
-- Future: components, component_types, component_instances, mep_systems, connections
```
- `data BLOB` is CBOR-encoded per-type payload (geometry + properties).
- Save is incremental: only dirty rows rewritten in a single transaction. Atomic write via SQLite journal.
- Load is streamed by layer to keep UI responsive on large projects.

### `cad/tools` — ITool interface
```cpp
class ITool {
public:
    virtual void onActivate(ToolContext&) = 0;
    virtual void onPointerMove(QPointF worldPos, ModKeys) = 0;
    virtual void onPointerDown(QPointF worldPos, MouseButton, ModKeys) = 0;
    virtual void onKey(QKeyEvent*) = 0;
    virtual void onDeactivate() = 0;
    virtual std::string id() const = 0;
};
```
- Tools live in `cad/tools/`: `LineTool`, `PolylineTool`, `RectangleTool`, `CircleTool`, `ArcTool`, `TextTool`, `DimensionTool`, `MoveTool`, `CopyTool`, `RotateTool`, `ScaleTool`, `MirrorTool`, `TrimTool`, `ExtendTool`, `OffsetTool`, `FilletTool`, `SelectTool`.
- Tools cooperate with `snap/` to resolve cursor position into snapped world coords.

### `snap/` — Snap engine
- Snap candidates queried from R-tree within a screen-space radius around the cursor.
- Snap kinds in v1: grid, endpoint, midpoint, center, ortho (constrains delta to 0/90°).
- Visual indicator widget overlays on viewport (small colored marker per snap kind).
- Per-mode toggle in status bar; ortho on F8, grid on F7 (AutoCAD-familiar).

### `ui/ribbon/` — Custom ribbon
- `RibbonBar` (top widget) contains tabs.
- `RibbonTab` contains horizontally-arranged `RibbonPanel`s.
- `RibbonPanel` contains `RibbonButton`s (large + small) and other controls.
- v1 tabs: **Home** (draw + modify), **Annotate** (text + dimensions), **View** (zoom/pan/snap toggles), **Manage** (layers, units, settings).
- Contextual tabs appear when relevant (e.g., **Text Edit** when text tool active).

### `mep/` — Scaffold + TestSymbol
- Interfaces: `IMepElement`, `IConnector` (point + direction + system kind), `ISystemType`, `IComponentDefinition`, `IComponentType`, `IComponentInstance`.
- `TestSymbol` — a concrete component that:
  - Has a hard-coded definition (a labeled circle with 2 connectors).
  - Can be placed via a ribbon button ("Test Symbol").
  - Owns generated `CadPolyline` + `CadText` primitives.
  - When user clicks any generated primitive, selection resolves up to the `TestSymbol` (proves `OwnerElementId` works).
  - Selectable, movable, deletable — proves the full lifecycle.

---

## Implementation Phases (for the impl agent)

Build in this order. Each phase ends with a runnable, demoable artifact.

1. **Phase 0 — Bootstrap**
   - Create public GitHub repo, push initial scaffold, GPL-3.0 LICENSE file, README with build instructions, CONTRIBUTING.md.
   - CMake, vcpkg manifest, GitHub Actions CI for Win + Linux, basic Qt window that opens.
   - Acceptance: `cmake --build` succeeds on both OSes; empty window opens; CI green on first push.

2. **Phase 1 — Core engine (no UI)**
   - `core/geometry`, `core/entity`, `core/command`, `core/spatial`, `core/document`.
   - Unit tests for each.
   - Acceptance: GTest suite green; can create a Document, add entities, query R-tree, undo/redo.

3. **Phase 2 — Render pipeline**
   - `render/` + `ui/DocumentWindow` showing a `ViewportRenderer`.
   - Pan/zoom working. Hardcoded test entities rendered. Golden-image baselines committed.
   - Acceptance: load 100k synthetic line entities → 60fps pan/zoom verified with FPS counter overlay.

4. **Phase 3 — Drawing tools + snap**
   - `cad/tools/*`, `snap/`, status bar, tool activation from ribbon stubs.
   - Acceptance: draw lines/polylines/rects/circles/arcs/text/dimensions with snap. Modify (move/copy/rotate/scale/mirror/trim/extend/offset/fillet) all functional.

5. **Phase 4 — Layers + properties + persistence**
   - `LayersPalette`, `PropertiesPalette`, `io/ProjectStore`, save/load `.vfproj`.
   - Acceptance: save a project, close, reopen — exact reproduction. Layers manipulable via palette.

6. **Phase 5 — MDI shell + ribbon polish**
   - Multiple `.vfproj` open in tabs, full ribbon (Home/Annotate/View/Manage), contextual tabs.
   - Acceptance: multi-document workflow smooth; ribbon feels Revit-like.

7. **Phase 6 — MEP/Component scaffold + TestSymbol**
   - All interfaces in `mep/`, `TestSymbol` placeable from ribbon, owner-id selection upchain.
   - Acceptance: place TestSymbol, click its generated geometry, see the TestSymbol selected (not the raw primitive). Move/delete TestSymbol moves/deletes its generated graphics atomically.

8. **Phase 7 — Hardening**
   - Perf benchmarking against 100k-entity target.
   - Golden-image render regression coverage for all entity types.
   - Crash-free 1-hour soak test.

---

## v1 Acceptance Criteria

A user can:
- [ ] Launch the app on Windows or Linux.
- [ ] Create a new `.vfproj`, save it, close, reopen — exact state restored.
- [ ] Open multiple projects in MDI tabs simultaneously.
- [ ] Draw lines, polylines, rectangles, circles, arcs, text, dimensions (linear/aligned/angular).
- [ ] Modify entities: move, copy, rotate, scale, mirror, trim, extend, offset, fillet.
- [ ] Snap to grid, endpoint, midpoint, center; lock to ortho.
- [ ] Manage layers (create, rename, recolor, hide, lock) via Layers palette.
- [ ] Edit any entity's properties via Properties palette.
- [ ] Undo and redo every action via `Ctrl+Z` / `Ctrl+Y` / ribbon.
- [ ] Pan and zoom smoothly at 60fps in a project with 100k entities.
- [ ] Place a TestSymbol (MEP example), select it by clicking its generated geometry, move and delete it.
- [ ] Project load time <2s for a 100k-entity file.

Engineering:
- [ ] `src/core/` has no Qt dependencies.
- [ ] All commands go through `CommandStack` — no direct document mutation from tools.
- [ ] GTest unit suite + golden-image render tests run green in CI on Windows and Linux.
- [ ] No memory leaks under 1-hour soak test (verified by Valgrind on Linux build).
- [ ] Repo public on GitHub with GPL-3.0 LICENSE, README, CONTRIBUTING.md.

---

## Verification (for the human reviewing the impl)

End-to-end verification, executable steps:

```powershell
# Build
cmake -S . -B build -DCMAKE_TOOLCHAIN_FILE=$env:VCPKG_ROOT\scripts\buildsystems\vcpkg.cmake
cmake --build build --config Release

# Run unit + render tests
ctest --test-dir build --output-on-failure

# Manual: run the app
.\build\src\app\Release\volmaris-forge.exe

# Perf check: open the synthetic 100k-entity stress project (ships in tests/data/stress_100k.vfproj)
# Verify FPS overlay shows >=60fps during continuous pan/zoom.
```

---

## Critical Files to Create First (priority order)

1. `LICENSE` (GPL-3.0 text), `README.md`, `CONTRIBUTING.md`
2. `CMakeLists.txt`, `vcpkg.json`, `.github/workflows/ci.yml` — get CI green on empty project
3. `src/core/geometry/Vec2.h`, `Bbox.h` — foundation types
4. `src/core/entity/EntityStore.{h,cpp}` + tests
5. `src/core/command/CommandStack.{h,cpp}` + tests
6. `src/core/spatial/RTreeIndex.{h,cpp}` + tests
7. `src/render/ViewportRenderer.{h,cpp}` — first visible rendering
8. `src/cad/entities/CadLine.{h,cpp}` + `LineTool` — first end-to-end drawing
9. `src/io/ProjectStore.{h,cpp}` — first save/load round-trip
10. `src/mep/interfaces/IMepElement.h` + `src/mep/test_symbol/TestSymbol.{h,cpp}` — proves layered architecture

Everything else hangs off these.

---

## Out of Scope for v1 (explicit)

- Real MEP elements beyond TestSymbol (pipes, ducts, pumps, valves, etc.)
- Calculations (flow, pressure drop, validation)
- Library import/export beyond stubbing
- DWG/DXF/IFC/PDF interop
- 3D anything
- Hatches, manual blocks, arrays (deferred from v1 — re-add in v2 if needed)
- Macros / scripting (Python or otherwise)
- Collaboration / multi-user
- macOS build
- Linetype / lineweight on layers
- Advanced snaps (intersection, perpendicular, tangent, nearest, polar, object tracking)

These are deliberately deferred. The architecture is set up to accept them without rework. See `00-vision.md` for the long-term direction on what eventually grows into the program.
