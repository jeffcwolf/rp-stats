# rp-stats — Specification

> **Version:** 0.1.0 (initial draft)
> **Date:** 2026-03-19
> **Status:** Draft — awaiting human sign-off

---

## Changelog

| Date | Change |
|---|---|
| 2026-03-19 | Initial specification created from Phase 0 brainstorming |

---

## Problem Statement

A developer maintaining multiple Rust projects in a single directory has
no quick way to see the health and shape of their portfolio at a glance.
Answering basic questions — which projects have tests, which have CI, how
large is each codebase, when was each last touched — requires opening each
project individually. rp-stats solves this by scanning a directory of Rust
projects, collecting per-project statistics from the filesystem and
Cargo.toml, and producing both a machine-readable JSON registry and a
human-readable HTML dashboard.

## User Persona

A solo or small-team Rust developer who maintains 5–50 Rust projects (or
workspace members) in a parent directory. They want a periodic snapshot of
their project portfolio — not continuous monitoring, not CI integration,
just a local CLI they run when they want to see the big picture. They are
comfortable with the command line and do not need a GUI beyond the
generated HTML file opened in a browser.

---

## Commands / Features

### CLI Interface

rp-stats is a single-command CLI tool:

```
rp-stats <target-dir> [OPTIONS]
```

**Arguments:**

| Argument | Required | Description |
|---|---|---|
| `<target-dir>` | Yes | The directory to scan for Rust projects |

**Options:**

| Flag | Default | Description |
|---|---|---|
| `--output-json <path>` | `./rp-stats.json` | Path for the JSON registry output |
| `--output-html <path>` | `./rp-stats.html` | Path for the HTML dashboard output |
| `--json-only` | false | Produce only the JSON registry, skip HTML |
| `--html-only` | false | Produce only the HTML dashboard, skip JSON |
| `--recursive` | false | Recurse into subdirectories to find projects (default: immediate children only) |

**Notes:**
- `--json-only` and `--html-only` are mutually exclusive.
- Default output paths are relative to the current working directory.
- When `--recursive` is set, the scanner skips `target/`, hidden
  directories (`.`-prefixed), and directories already identified as
  members of a detected workspace.

### Project Detection

A directory is a Rust project if it contains a `Cargo.toml` file. Detection
logic:

- **Default (no `--recursive`):** Check each immediate subdirectory of
  `<target-dir>` for a `Cargo.toml`.
- **With `--recursive`:** Walk the directory tree, skipping `target/` and
  hidden directories. When a `Cargo.toml` with `[workspace]` is found,
  its declared members are discovered and the scanner does not
  independently discover those member directories (no double-counting).

### Stats Collected Per Project

For each detected project, rp-stats collects:

| Stat | Source | Notes |
|---|---|---|
| Project name | `[package].name` in Cargo.toml | Falls back to directory name if `[package]` absent (workspace root without a package) |
| Workspace or single crate | Presence of `[workspace]` in Cargo.toml | Boolean |
| Number of member crates | `[workspace].members` in Cargo.toml | 0 for single crates; resolved glob patterns counted |
| Lines of Rust code | `.rs` files in `src/`, `tests/`, `benches/`, `examples/` | Non-blank lines only |
| Number of source files | `.rs` files in the same directories | Count |
| Number of `#[test]` functions | Regex scan of `.rs` files | Matches `#[test]` attribute only (not `#[tokio::test]` or custom macros) |
| Direct dependency count | `[dependencies]` section of Cargo.toml | Runtime dependencies only; `[dev-dependencies]` and `[build-dependencies]` excluded |
| Has `tests/` directory | Filesystem check | Boolean |
| Has `.github/workflows/` | Filesystem check | Boolean |
| Last git commit date | `git log -1 --format=%aI` equivalent via `.git` parsing | ISO 8601 format; `null` if not a git repo |
| Last git commit hash | `git log -1 --format=%H` equivalent via `.git` parsing | Full SHA; `null` if not a git repo |

**Git info without shelling out:** The last commit date and hash are read
by parsing `.git/HEAD` → ref → reading the commit object directly, or by
invoking `git log` as a subprocess. This is the one exception to the
"no external tools" constraint — git is assumed present on the system.
If `.git` does not exist in the project directory (or any parent up to
`<target-dir>`), the git fields are `null`.

### Workspace Handling

For workspace projects, rp-stats produces:

- **One summary row** for the workspace root, with stats aggregated
  across all member crates (total LOC, total files, total tests, total
  deps, etc.).
- **One row per member crate** with that crate's individual stats.

In the JSON output, member crate data is nested under the workspace
entry. In the HTML output, member rows are grouped under the workspace
row and **collapsed by default** — the user clicks to expand.

### JSON Registry Format

The JSON output is an object with metadata and a list of project entries:

```json
{
  "generated_at": "2026-03-19T14:30:00Z",
  "scanned_directory": "/home/user/code",
  "project_count": 12,
  "projects": [
    {
      "name": "my-project",
      "path": "/home/user/code/my-project",
      "is_workspace": false,
      "crate_count": 1,
      "rust_lines": 1450,
      "source_files": 12,
      "test_count": 34,
      "dependency_count": 8,
      "has_tests_dir": true,
      "has_ci": true,
      "last_commit_date": "2026-03-15T10:22:00+00:00",
      "last_commit_hash": "a1b2c3d4...",
      "members": null
    },
    {
      "name": "my-workspace",
      "path": "/home/user/code/my-workspace",
      "is_workspace": true,
      "crate_count": 3,
      "rust_lines": 5200,
      "source_files": 38,
      "test_count": 112,
      "dependency_count": 15,
      "has_tests_dir": true,
      "has_ci": true,
      "last_commit_date": "2026-03-18T16:45:00+00:00",
      "last_commit_hash": "e5f6a7b8...",
      "members": [
        {
          "name": "core",
          "path": "crates/core",
          "rust_lines": 2800,
          "source_files": 18,
          "test_count": 65,
          "dependency_count": 5,
          "has_tests_dir": false
        }
      ]
    }
  ]
}
```

### HTML Dashboard

A single self-contained static HTML file (no external dependencies —
CSS and JavaScript are inlined). The dashboard contains:

**Summary bar** at the top showing aggregate stats:
- Total projects scanned
- Total lines of Rust code
- Total `#[test]` functions
- Total source files

**Project table** with columns for each stat. Features:
- **Column sorting:** Click any column header to sort ascending/descending.
- **Text filter:** A search box that filters rows by project name
  (substring match, case-insensitive).
- **Toggle filters:** Checkboxes for "Has CI," "Has tests/ dir," and
  "Is workspace" that show/hide matching rows.
- **Workspace grouping:** Workspace rows are expandable — click to
  reveal/collapse member crate rows beneath the workspace summary row.
  Collapsed by default.

**No external resources.** The HTML file must render correctly when opened
from the local filesystem (`file://` protocol) with no network access.

---

## Non-Feature List

These are explicitly excluded from v1 scope with rationale.

| Exclusion | Rationale |
|---|---|
| Async I/O | Scanning the local filesystem is I/O-bound but not latency-sensitive. Synchronous I/O is simpler and sufficient. |
| Caching or incremental scan | Full rescan each run. The target use case (5–50 projects) completes in seconds. Caching adds complexity without meaningful benefit at this scale. |
| `cargo metadata` or any external analysis tool | All stats are derived from filesystem walking and TOML parsing. This avoids requiring a Rust toolchain on the machine running rp-stats, and avoids slow compilation/resolution. Git is the only external tool used. |
| HTML template engine | The HTML output is one static file. A format string or simple builder is sufficient. A template engine is a dependency that provides no depth. |
| Custom test macro detection (`#[tokio::test]`, `#[rstest]`, etc.) | Matching `#[test]` only keeps the regex simple and predictable. Custom macros could be a v1.1 addition. |
| `[dev-dependencies]` and `[build-dependencies]` counting | Runtime dependency count is the most useful signal. Separate dev/build counts could be a v1.1 addition. |
| Watch mode or continuous monitoring | rp-stats is a run-once snapshot tool. Watching for filesystem changes is a different category of tool. |
| Remote repository scanning | Input is a local directory. Cloning or fetching remote repos is out of scope. |
| Configuration file | All options are CLI flags. A config file adds complexity for a tool with few options. |
| Multi-language support | Rust projects only. Detecting other languages (Go, Python, etc.) is a different tool. |
| Interactive TUI | Output is JSON + static HTML. A terminal UI adds a significant dependency surface for marginal benefit. |
| Per-file stats or detailed breakdowns | Stats are per-project (or per-crate for workspace members). File-level detail belongs in a different tool (e.g., tokei). |

---

## Success Checklist

Every box must be checked to declare v1.0. This is the definition of done.

### Functional

- [ ] `rp-stats <dir>` scans immediate subdirectories and produces both JSON and HTML output
- [ ] `--recursive` flag enables deep scanning with `target/` and hidden directory exclusion
- [ ] `--json-only` produces only JSON; `--html-only` produces only HTML
- [ ] `--output-json` and `--output-html` control output file paths
- [ ] Workspace detection: `[workspace]` in Cargo.toml triggers workspace handling
- [ ] Workspace member resolution: glob patterns in `[workspace].members` are resolved
- [ ] Member crate stats are collected individually and aggregated at the workspace level
- [ ] All 11 stats collected correctly per project (name, workspace flag, crate count, LOC, source files, test count, dep count, tests dir, CI dir, commit date, commit hash)
- [ ] LOC counts non-blank lines in `.rs` files only
- [ ] `#[test]` count matches the `#[test]` attribute via regex
- [ ] Dependency count reads `[dependencies]` section only
- [ ] Git info is `null` when `.git` is absent (not an error)
- [ ] Malformed `Cargo.toml` is skipped with a warning to stderr, scan continues
- [ ] JSON output matches the schema defined in this spec
- [ ] HTML dashboard renders correctly from `file://` with no network access
- [ ] HTML summary bar shows aggregate stats (total projects, LOC, tests, files)
- [ ] HTML table supports ascending/descending sort on every column
- [ ] HTML table has a text filter for project name
- [ ] HTML table has toggle filters for Has CI, Has tests dir, Is workspace
- [ ] HTML workspace rows are expandable/collapsible, collapsed by default

### Quality

- [ ] `cargo test --workspace` passes
- [ ] `cargo fmt --check` passes
- [ ] `cargo clippy --workspace -- -D warnings` passes
- [ ] `livery/bin/prism check . --strict` passes
- [ ] Property tests exist for TOML parsing and stat aggregation
- [ ] Integration tests invoke the binary via `assert_cmd` and verify JSON output structure
- [ ] Integration tests verify HTML output contains expected elements
- [ ] All public items have doc comments

---

## Risk Register

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R1 | **Workspace member glob resolution is complex.** Cargo supports glob patterns like `crates/*` in `[workspace].members`. Resolving these correctly (including nested globs, exclude patterns) requires careful implementation. | Medium | High — incorrect resolution means wrong project list | Implement basic glob matching (`*` only, no `**` or complex patterns). Document the limitation. Test with real-world workspace layouts. |
| R2 | **Git commit parsing without shelling out is fragile.** Reading `.git/HEAD`, following refs, and parsing commit objects handles only the simple case. Packed refs, shallow clones, and worktrees add edge cases. | Medium | Medium — git info may be `null` unexpectedly | Use `git log -1` as a subprocess (git is the one allowed external tool). Fall back to `null` on any failure. |
| R3 | **HTML expand/collapse adds JavaScript complexity.** Grouped workspace rows with toggle behavior require non-trivial JS in a single self-contained file. | Low | Medium — increases HTML generation complexity and testing surface | Keep the JS minimal (vanilla, no framework). Test expand/collapse behavior manually. Consider deferring to v1.1 if implementation time exceeds estimate. |
| R4 | **TOML parsing edge cases.** Real-world Cargo.toml files use inline tables, dotted keys, multi-line strings, and other TOML features. A naive parser will break. | Low | High — broken parsing means wrong or missing stats | Use a well-tested TOML parsing crate (`toml` or `toml_edit`). Do not write a custom parser. |
| R5 | **LOC counting in `target/` or generated files.** If `--recursive` mode fails to exclude `target/` directories or generated code, LOC counts will be wildly inflated. | Low | High — misleading data | Hardcode `target/` exclusion. Also exclude common generated directories (`out/`, `.cargo/`). Test with a project that has a populated `target/`. |
| R6 | **Large directory trees with `--recursive`.** A user points rp-stats at `~/code` containing hundreds of nested projects with large `target/` directories. Even with exclusions, the walk could be slow. | Low | Low — tool is slow but correct | Exclusion of `target/` and hidden dirs mitigates the main cost. No further optimization needed for v1. |

---

## Non-Negotiable Constraints

These constraints are architectural and must not be violated.

1. **Rust projects only.** Detection is via `Cargo.toml`. No other project
   types are scanned or reported.
2. **No external analysis tools.** All stats are derived from filesystem
   walking and TOML parsing. No `cargo metadata`, no `cargo test --list`.
   Git (as a subprocess) is the sole exception for commit info.
3. **No async.** Synchronous I/O throughout. The tool scans local
   directories.
4. **No caching.** Full rescan each run.
5. **Self-contained HTML.** The HTML output is a single file with all CSS
   and JS inlined. No CDN links, no external fonts, no images that
   require network access.
6. **JSON is canonical.** The JSON registry is the primary data format.
   The HTML dashboard is a rendering of the same data. If both are
   generated in one run, they must reflect identical data.
