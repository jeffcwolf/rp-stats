# rp-stats — Architecture

> **Version:** 0.1.0
> **Date:** 2026-03-19
> **Status:** Draft

---

## Overview

rp-stats is a single-binary CLI tool. It scans a directory of Rust projects,
collects per-project statistics from the filesystem and Cargo.toml files,
and produces a JSON registry and/or an HTML dashboard. The architecture is a
linear pipeline: discover → collect → aggregate → emit.

---

## Crate Structure

Single crate, single binary. No library crate, no workspace. The project is
small enough that crate-level decomposition would be premature. All module
boundaries are internal (`pub(crate)`).

```
src/
  main.rs          — CLI entry point: argument parsing, orchestration
  cli.rs           — Argument definition and parsing (clap)
  discovery.rs     — Locate Rust projects in the target directory
  collector.rs     — Collect all stats for a single project
  cargo_toml.rs    — Parse Cargo.toml for project metadata and dependencies
  line_count.rs    — Count non-blank lines and source files in .rs files
  test_count.rs    — Count #[test] attributes via regex
  git_info.rs      — Retrieve last commit date and hash via git subprocess
  registry.rs      — The ProjectEntry data model and JSON serialization
  html.rs          — Generate the self-contained HTML dashboard
```

---

## Module Responsibilities

### `main.rs` — Orchestration

Parses CLI arguments, calls discovery, iterates over discovered projects to
collect stats, and passes the completed registry to the output emitters.
Contains no business logic — only sequencing.

### `cli.rs` — CLI Argument Parsing

Defines the CLI interface using `clap`. Produces a validated `Config` struct
consumed by `main`. Owns the `Config` type.

**Hides:** clap dependency, argument validation, default path resolution.

### `discovery.rs` — Project Discovery

Given a target directory and a recursive flag, returns a list of discovered
Rust project paths. A Rust project is a directory containing `Cargo.toml`.

In non-recursive mode: checks immediate subdirectories only.
In recursive mode: walks the tree, skipping `target/`, hidden directories,
and workspace member directories already claimed by a parent workspace.

**Hides:** directory traversal logic, exclusion rules, workspace member
deduplication.

**Key type:** `DiscoveredProject` — carries the path and whether the
Cargo.toml contains a `[workspace]` section (detected during discovery to
avoid re-reading the file).

### `collector.rs` — Per-Project Stat Collection

Given a project path, produces a complete `ProjectEntry` (or workspace
entry with member entries). Coordinates calls to `cargo_toml`, `line_count`,
`test_count`, and `git_info`. For workspace projects, resolves member
directories from `[workspace].members` glob patterns, collects stats for
each member, and aggregates totals for the workspace summary row.

**Hides:** the coordination between sub-collectors, workspace member
resolution and glob expansion, aggregation arithmetic.

### `cargo_toml.rs` — Cargo.toml Parsing

Reads and parses a Cargo.toml file. Extracts: package name, whether
`[workspace]` is present, workspace member patterns, and runtime dependency
count from `[dependencies]`.

**Hides:** TOML parsing (uses the `toml` crate), key lookup paths, handling
of missing or malformed sections.

### `line_count.rs` — Rust Line and File Counting

Walks `src/`, `tests/`, `benches/`, `examples/` within a project directory.
Counts non-blank lines in `.rs` files and the number of `.rs` files.

**Hides:** directory walking, blank-line detection, which subdirectories
are included.

### `test_count.rs` — Test Function Counting

Scans `.rs` files for `#[test]` attributes using regex. Returns the total
count for a project directory.

**Hides:** regex pattern, file traversal, which directories are scanned.

### `git_info.rs` — Git Commit Info

Runs `git log -1 --format=...` as a subprocess in the project directory.
Returns the last commit date (ISO 8601) and full SHA hash, or `None` if
the project is not a git repository or git is not available.

**Hides:** subprocess execution, output parsing, all failure modes (returns
`None` on any error).

### `registry.rs` — Data Model and JSON Output

Defines `ProjectEntry`, `MemberEntry`, and `Registry` — the canonical data
types for collected stats. Serializes `Registry` to JSON using `serde_json`.

**Hides:** serde annotations, JSON formatting, the registry metadata
(generated_at timestamp, scanned directory, project count).

### `html.rs` — HTML Dashboard Generation

Takes a `Registry` and produces a self-contained HTML string. Inlines all
CSS and JavaScript. The HTML includes a summary bar, a sortable/filterable
project table, and CSS-based workspace member grouping.

**Hides:** HTML structure, CSS styles, inline JavaScript for sorting and
filtering, workspace row grouping logic.

---

## Data Flow

```
CLI args
  │
  ▼
discovery::discover(target_dir, recursive)
  │
  ▼
Vec<DiscoveredProject>
  │
  ▼
collector::collect(project) ──for each──► ProjectEntry
  │                                         │
  ├── cargo_toml::parse(path)               │
  ├── line_count::count(path)               │
  ├── test_count::count(path)               │
  └── git_info::last_commit(path)           │
                                            ▼
                                     Registry { projects: Vec<ProjectEntry> }
                                            │
                              ┌─────────────┼─────────────┐
                              ▼             ▼             ▼
                         registry.json  dashboard.html  (both)
```

---

## Key Design Decisions

### Single crate, no library

The tool is a standalone CLI with no reuse need. A library crate would add
a public API surface with no caller. If reuse emerges later, extraction is
straightforward because module boundaries are clean.

### Git via subprocess, not .git parsing

Parsing `.git/HEAD`, following refs, and reading commit objects handles only
the simple case. Packed refs, shallow clones, and worktrees add edge cases
that are already solved by the `git` binary. The spec explicitly permits
git as the one allowed external tool.

### No `cargo metadata`

All stats are derived from filesystem walking and TOML parsing. This avoids
requiring a full Rust toolchain on the machine running rp-stats, avoids
slow dependency resolution, and keeps the tool fast and self-contained.

### Workspace members as nested data, not top-level entries

In the JSON registry, workspace members are nested under their workspace
entry. In the HTML table, member rows are visually grouped beneath the
workspace summary row. This prevents double-counting and preserves the
workspace-member relationship in both output formats.

### Synchronous I/O throughout

The tool scans local directories. The bottleneck is filesystem metadata
calls, not network latency. Async would add complexity (a runtime
dependency, colored functions) with no performance benefit.

---

## Dependencies

| Crate | Purpose |
|---|---|
| `clap` | CLI argument parsing |
| `toml` | Cargo.toml parsing |
| `serde` + `serde_json` | Registry serialization |
| `chrono` | Timestamp formatting for `generated_at` |
| `regex` | `#[test]` attribute matching |
| `glob` | Workspace member pattern resolution |
| `walkdir` | Recursive directory traversal |
| `anyhow` | Error handling (binary crate) |

---

## Error Strategy

Binary crate: `anyhow` for top-level error propagation. Individual
collectors return `Option` or `Result` where partial failure is expected
(e.g., git info is `None` for non-git projects, malformed Cargo.toml is
skipped with a stderr warning). The tool does not abort on a single
project's failure — it logs a warning and continues scanning.

---

## Testing Strategy

- **Unit tests** in each module for parsing, counting, and edge cases.
- **Property tests** for TOML parsing (arbitrary valid/invalid inputs)
  and stat aggregation (sum invariants across workspace members).
- **Integration tests** in `tests/` using `assert_cmd` to invoke the
  binary against fixture directories and verify JSON output structure
  and HTML content.
