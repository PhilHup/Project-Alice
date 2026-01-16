# PROJECT KNOWLEDGE BASE

**Generated:** 2026-01-15T23:00:00Z
**Commit:** a90dd964a
**Branch:** main

## OVERVIEW
Project Alice — large C++ codebase (CMake) that implements a Victoria‑2 retro clone. Primary concerns: game engine, data parsers, AI, and platform entry points. The repo contains vendored third‑party code (zstd) and multiple subtools (SaveEditor, Launcher, ParserGenerator).

## STRUCTURE (non-obvious only)
```
./
├── src/                 # Main C++ source tree (platform entry points, engine, ai, common types)
├── tests/               # Unit & integration tests (CMake test targets)
├── SaveEditor/          # Separate executable for save editing
├── Launcher/            # Launcher tooling
├── docs/                # Project docs (user/developer)
├── libs/                # External libs and submodules
└── .github/workflows/   # CI (includes opencode workflow)
```

## WHERE TO LOOK (common tasks)
- Build & iterate: src/, CMakeLists.txt, CMakePresets.json
- Entry points: src/entry_point_win.cpp, src/entry_point_nix.cpp
- AI: src/ai/ (ai_*.cpp / .hpp)
- Core types: src/common_types/
- Compression/Vendored: src/zstd/ (do not edit unless upstream sync intended)
- Tests: tests/ (CMakeLists and test_*.cpp)
- Tools: SaveEditor/, Launcher/, ParserGenerator/

## COMMANDS
```bash
# Configure & build (out-of-tree, recommended)
cmake -S . -B build -G "Ninja" -DCMAKE_BUILD_TYPE=RelWithDebInfo
cmake --build build -j$(nproc)
# Run tests
cmake --build build --target test
# Run a single binary
./build/src/AliceExecutable  # replace with real target name
```

## CONVENTIONS & GOTCHAS
- CMake is the canonical build system; prefer editing CMakePresets.json for dev presets.
- Platform-specific code is split into entry_point_win.cpp and entry_point_nix.cpp; avoid duplicating platform logic across files.
- zstd under src/zstd is vendored; treat it as third‑party — do not refactor lightly.
- Tests are CMake targets under tests/; running `ctest` is the canonical way to run them.

## ANTI-PATTERNS (project-specific)
- Avoid large cross-cutting header changes; they cause long rebuilds.
- Don’t modify vendored code in src/zstd; instead update upstream and vendor a new version.

## QUICK CODE MAP (high‑level)
- Entry points: src/entry_point_*.cpp
- Core types & utilities: src/common_types/
- AI modules: src/ai/ (ai_war.cpp, ai_focuses.cpp, ai_economy.cpp)
- Compression: src/zstd/

## NOTES
- CI includes an opencode workflow (.github/workflows/opencode.yml) which reads OPENCODE_API_KEY from repository secrets.
- Large repo: expect long scan/build times. Use incremental builds and ccache where supported.

<!-- End of AGENTS.md -->