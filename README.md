# Volmaris Forge

A high-performance 2D HVAC schematic editor. Open source under **GPL-3.0**.

> **Status:** early bootstrap (Phase 0). The drawing engine and MEP semantic layer are being built out per [`docs/01-architecture.md`](docs/01-architecture.md).

## What it is

Volmaris Forge is a ground-up CAD application focused on HVAC schematics. It combines a fast custom drawing engine (OpenGL via Qt 6) with a Revit-style component/family system for MEP elements. The CAD core ships first; the MEP layer is scaffolded with one working example (`TestSymbol`) and grows from there.

Locked technical choices: **C++20, Qt 6 LTS (LGPL), OpenGL renderer, CMake + vcpkg, Boost.Geometry R-tree, SQLite project files, GPL-3.0.** See [`docs/01-architecture.md`](docs/01-architecture.md) for the full build contract.

## Building from source

### Prerequisites

- **CMake** ≥ 3.24
- A C++20 compiler:
  - Windows: Visual Studio 2022 (MSVC 19.30+)
  - Linux: GCC 11+ or Clang 14+
- **vcpkg** with `$VCPKG_ROOT` set to its install path
- **Ninja** (recommended) or your platform's default generator

Dependencies (Qt 6, Boost.Geometry, SQLite, GTest) are pulled in automatically by the vcpkg manifest in this repo.

### Windows (PowerShell)

```powershell
cmake -S . -B build -DCMAKE_TOOLCHAIN_FILE="$env:VCPKG_ROOT\scripts\buildsystems\vcpkg.cmake"
cmake --build build --config Release
.\build\src\app\Release\volmaris-forge.exe
```

### Linux

```bash
cmake -S . -B build -DCMAKE_TOOLCHAIN_FILE="$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake" -DCMAKE_BUILD_TYPE=Release
cmake --build build
./build/src/app/volmaris-forge
```

### Running tests

```bash
ctest --test-dir build --output-on-failure
```

## Repository layout

See [`docs/01-architecture.md`](docs/01-architecture.md) §Repository Layout. Top-level summary:

- `src/core/` — pure C++ engine (no Qt deps)
- `src/cad/` — CAD entities + tools
- `src/mep/` — MEP / Component scaffold
- `src/render/` — OpenGL render pipeline
- `src/io/` — SQLite-backed project and library persistence
- `src/ui/` — Qt widgets (MDI shell, ribbon, palettes)
- `src/app/` — `main.cpp`
- `tests/` — GTest unit tests and golden-image render tests

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md).

## License

GPL-3.0-or-later. See [`LICENSE`](LICENSE).

Volmaris Forge links Qt 6 dynamically under the LGPL. SQLite is public domain. Boost is BSL-1.0. GoogleTest is BSD-3-Clause. All chosen runtime dependencies are GPL-3.0-compatible.
