# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

Rust workspace, edition 2024, MSRV 1.88 (`rust-version` in the root `Cargo.toml`; let-chains are used, so older toolchains fail).

```shell
cargo build --release                       # binary: ./target/release/vrp-cli
cargo fmt --all -- --check                  # rustfmt: max_width=120, use_small_heuristics=Max
cargo clippy --locked --workspace --all-features --all-targets -- -D warnings   # CI lint, warnings are errors
cargo test --locked --workspace --benches   # CI test run (test profile uses opt-level=1)

cargo test -p vrp-core <name_filter>        # single test / module, e.g. `cargo test -p vrp-pragmatic can_use_break`
cargo test --release -- --ignored discovery # long-running property/discovery tests (#[ignore]d; manual CI only)

cargo run --example cvrp                    # also: pdptw, custom_constraint, custom_objective (vrp-core/examples, run in CI)
./target/release/vrp-cli solve solomon examples/data/scientific/solomon/C101.100.txt --log
./solve_problem.sh examples/data/pragmatic/objectives/berlin.default.problem.json   # pragmatic, writes solution + geojson next to input, runs --check
```

Language bindings (Python via maturin/pyo3 `py_bindings` feature, WebAssembly via wasm-pack) consume generated models. Generated output is **not committed**; regenerate after changing any pragmatic format or config type:

```shell
pip install -r vrp-cli/bindings/python/requirements-codegen.txt
npm ci --prefix vrp-cli/bindings/typescript
./vrp-cli/bindings/generate.sh   # runs `schema_gen` example (needs `schema` feature) -> JSON schemas -> pydantic + TS types
```

## Architecture

Crates, layered bottom-up:

- **`rosomaxa`** — problem-agnostic metaheuristic framework: evolution loop, populations (incl. the ROSOMAXA GSOM-based population), hyper-heuristics (`hyper/static_selective.rs`, `hyper/dynamic_selective.rs`), termination criteria. Knows nothing about VRP.
- **`vrp-core`** — VRP domain + solver built on rosomaxa.
  - `models/` — `problem` (jobs, fleet, costs), `solution` (routes/tours/activities), `common`. Problem/job/vehicle data is extended via typed **dimensions** rather than new struct fields.
  - `models/goal.rs` — central extension point. A `Feature` bundles an optional `FeatureConstraint` (hard constraints, returns `ConstraintViolation` with a `ViolationCode`), `FeatureObjective` (soft cost), and `FeatureState` (cached route/solution state). `GoalContext` composes features into a lexicographic/hierarchical goal.
  - `construction/features/` — built-in features (capacity, breaks, reloads, recharge, skills, groups, tour limits, work balance, transport cost, …), each typically exposed via a `*FeatureBuilder` or `create_*_feature`.
  - `construction/heuristics/` — insertion evaluation and insertion contexts; `construction/enablers/` — shared helpers features rely on (e.g. `FeatureCombinator`, multi-trip support).
  - `solver/` — the VRP-specific metaheuristic: `search/` holds ruin, recreate, local search operators, and composite searches (decompose, infeasible, redistribute, …) that rosomaxa's hyper-heuristic selects between. `solver/heuristic.rs` assembles the default configuration.
- **`vrp-scientific`** — readers for benchmark formats (Solomon, Li&Lim, TSPLIB).
- **`vrp-pragmatic`** — the JSON "pragmatic" format used by real-world callers. `format/problem/*_reader.rs` translate JSON into core models (`goal_reader.rs` maps JSON objectives to core features), `validation/` validates input before solving, `checker/` independently verifies a solution's feasibility (used by `--check`).
- **`vrp-cli`** — CLI (`commands/` = clap wiring, `extensions/` = the logic for solve/check/import/analyze/generate) and the library behind all foreign bindings. `src/interop/` holds the thin binding adapters (`c_api.rs`, `python.rs`, `wasm.rs`) over one shared JSON contract (`json.rs`). Many deps are `cfg(not(target_arch = "wasm32"))`, so wasm-only and native-only code paths must both compile.
- `experiments/heuristic-research` — research harness/visualizer for heuristics; `examples/json-pragmatic` — format examples.

Adding a new VRP constraint/objective usually means: implement a `Feature` in `vrp-core/src/construction/features/`, then (for the pragmatic format) add model fields in `vrp-pragmatic/src/format/problem/model.rs`, wire them in the relevant reader/`goal_reader.rs`, add validation and a checker rule, and add feature tests under `vrp-pragmatic/tests/features/`. Format type changes affect the generated bindings.

## Test layout conventions

Unit tests do **not** live inline or in cargo's integration `tests/` sense. Each crate keeps them under `<crate>/tests/unit/` mirroring `src/`, and the source file pulls them in:

```rust
#[cfg(test)]
#[path = "../../tests/unit/models/goal_test.rs"]
mod goal_test;
```

Shared test helpers live in `<crate>/tests/helpers/` and are mounted from `lib.rs` as a `#[cfg(test)]` module (with `#[macro_use]`). `vrp-pragmatic` additionally mounts `tests/features/` (scenario tests per feature), `tests/generator/` + `tests/discovery/` (proptest-style generated problems, `#[ignore]`d), and `tests/regression/`. When adding a test for `src/foo/bar.rs`, put it at `tests/unit/foo/bar_test.rs` and add the `#[path]` module to `bar.rs`.

## Other notes

- `CHANGELOG.md` keeps an `## [Unreleased]` section; user-visible changes are recorded there.
- `master` contains unreleased changes; breaking changes can occur in minor versions.
- User docs (mdBook) are in `docs/`.
