# Contributing to Volmaris Forge

Thanks for considering a contribution. Volmaris Forge is open source under GPL-3.0; by submitting a contribution you agree it will be licensed under the same terms.

## Ground rules

- **Read [`docs/01-architecture.md`](docs/01-architecture.md) first.** It is the authoritative build contract. Decisions there are locked unless explicitly reopened.
- **`docs/00-vision.md` is forward-looking only.** Don't pull features from it into a PR unless they've been promoted into `01-architecture.md`.
- **`src/core/` must stay Qt-free.** This is load-bearing for testability. Pure C++ only.
- **All commands flow through `CommandStack`.** Tools build commands and push them — they never mutate the document directly.

## Development workflow

1. Fork and branch from `main`. Branch names: `feat/<topic>`, `fix/<topic>`, `chore/<topic>`, `docs/<topic>`.
2. Build & test locally (see [`README.md`](README.md)) before pushing.
3. Open a PR against `main` with a clear description of *what* and *why*.
4. CI must be green on both Windows and Linux.

## Coding style

- **C++20.** Use modern facilities (`std::span`, `std::optional`, structured bindings) where they aid clarity.
- **Formatting:** `clang-format` with the repo's `.clang-format` (LLVM-based). Run before committing.
- **Naming:** `PascalCase` for types, `camelCase` for functions/methods, `snake_case` for local variables. `m_` prefix for private members. Constants `kPascalCase`.
- **Headers:** `#pragma once`. Forward-declare in headers where possible.
- **No raw `new`/`delete`** outside low-level resource wrappers. Use `std::unique_ptr` / `std::shared_ptr`.

## Commit messages

[Conventional Commits](https://www.conventionalcommits.org/). Examples:

- `feat(core): add EntityStore dirty-event emission`
- `fix(render): correct line-strip batching on zoom`
- `chore(ci): bump vcpkg baseline`
- `docs(architecture): clarify snap candidate radius`
- `test(spatial): cover R-tree update path`

## Testing

- **Unit tests** (`tests/unit/`) — GTest. Required for all `src/core/` changes.
- **Render tests** (`tests/render/`) — golden-image diffs against committed PNG baselines. Required when changing render output.
- `ctest --test-dir build --output-on-failure` must pass locally before opening a PR.

## Reporting issues

File issues on the GitHub tracker. Include:

- OS + compiler version
- CMake + vcpkg versions
- Minimal reproduction steps
- Expected vs. actual behavior
