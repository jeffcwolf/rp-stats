> **Extends:** `livery/CLAUDE-base.md` — read that file first, then this one.
>
> **Runtime:** This project uses Livery with Superpowers as the execution
> engine. See `livery/adapter-superpowers.md` for the integration.
> Superpowers skills handle workflow execution. CLAUDE-base.md is the
> design constitution and overrides any conflicting Superpowers guidance.

---

# rp-stats — Project Constitution

## What This Project Is

rp-stats is a CLI tool that scans a directory of Rust projects, collects
per-project statistics, writes the results to a JSON registry file, and
generates a single static HTML dashboard with basic sorting and filtering.

## Reference Documents

| Document | Load when |
|---|---|
| `livery/CLAUDE-base.md` | Every session (always loaded first) |
| `livery/WORKFLOW.md` | Understanding the phase sequence |
| `livery/adapter-superpowers.md` | Understanding how Superpowers integrates with Livery |
| `livery/standards/ousterhout.md` | Designing or reviewing any module |
| `livery/standards/readable-code.md` | Writing or reviewing names, comments, control flow |
| `livery/standards/rust-specifics.md` | Writing any Rust type, trait, error, or test |
| `SPEC.md` | Scope questions; checking whether a feature is in scope |
| `ARCHITECTURE.md` | Designing or modifying any module, crate, or public API |
| `SESSIONS.md` | Starting a session (read last 2–3 entries); ending a session |

## Stats Collected Per Project

- Project name (from Cargo.toml)
- Workspace or single crate
- Number of crates (if workspace)
- Lines of Rust code (.rs files)
- Number of source files
- Number of `#[test]` functions
- Direct dependency count (from Cargo.toml)
- Whether `tests/` directory exists
- Whether `.github/workflows/` exists
- Last git commit date and hash

## Project Constraints

- Rust projects only (detect via Cargo.toml)
- No external analysis tools — all stats derived from filesystem and basic parsing
- No async — synchronous I/O is sufficient for local filesystem scanning
- No caching or incremental mode — full rescan each run
- HTML output is a single self-contained static file
- JSON registry is the canonical data format

## Session Contract Commands

```bash
# Validation (every session)
cargo test --workspace
cargo fmt --check
cargo clippy --workspace -- -D warnings

# Quality gate (every session)
livery/bin/prism check . --strict
livery/bin/prism stats . --json
```

## Anti-Patterns Specific to This Project

- **Shelling out to cargo for stats.** All stats must be derived from
  filesystem walking and TOML parsing. No `cargo metadata`, no `cargo test --list`.
- **HTML template engines for a single file.** The HTML output is one
  static file. A format string or simple builder is sufficient.
- **Async for local filesystem operations.** This tool scans local
  directories. Synchronous I/O is correct.