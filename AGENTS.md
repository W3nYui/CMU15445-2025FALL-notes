# Repository Guidelines

## Project Structure & Module Organization
BusTub is a C++17 educational database system built with CMake. Core code lives in `src/`, with public headers under `src/include/` and implementations grouped by subsystem such as `buffer/`, `execution/`, `storage/`, `planner/`, and `optimizer/`. Tests live in `test/` and are organized by area, for example `test/execution/`, `test/storage/`, `test/txn/`, and SQL logic tests in `test/sql/`. Developer tools and local binaries are under `tools/`.

## Build, Test, and Development Commands
Use an out-of-tree build:

```bash
mkdir -p build
cd build
cmake -DCMAKE_BUILD_TYPE=Debug ..
make -j$(nproc)
```

Run all unit tests with `ctest --output-on-failure`. Run one test binary directly when iterating, for example `./test/txn_executor_test`. SQL logic regression tests are typically exercised through the shell tooling and `.slt` files in `test/sql/`. Formatting and lint targets are:

```bash
make format
make check-format
make check-lint
```

## Coding Style & Naming Conventions
Follow `.clang-format`: Google-based style, 2-space indentation, pointer alignment as `Type *ptr`, and a 120-column limit. Keep headers in `src/include/...` aligned with source files in `src/...`. Prefer descriptive PascalCase for classes, camelCase or snake_case to match surrounding code, and keep executor, plan, and page names consistent with existing modules.

## Testing Guidelines
Add or update focused tests for behavior changes. C++ tests use GoogleTest and usually follow `*_test.cpp` naming. SQL behavior should be covered with a matching `.slt` file when the change affects query execution. Before opening a PR, run the most relevant unit test binary plus `ctest --output-on-failure` if the change is broad.

## Commit & Pull Request Guidelines
Recent history favors short, task-oriented commit messages such as `完成Task3，实现了grace hash join...` or `完成aggregation算子`. Keep commits scoped to one logical change and mention the subsystem or task. PRs should include: what changed, why it changed, how it was tested, and any known limitations. Link the related assignment task or issue when applicable.

## Security & Configuration Tips
Do not publish solution code from private course work. Build from a `build/` subdirectory only; running CMake at the repo root is explicitly unsupported by this project. Prefer Linux or the course-supported environment when debugging sanitizer or concurrency issues.
