# Phase 0 — rp-stats Specification

## Before anything else

Read these documents in order:
1. `livery/CLAUDE-base.md`
2. `livery/WORKFLOW.md` — Phase 0 section
3. `livery/adapter-superpowers.md` — §4.1 (Phase 0 with Superpowers)
4. `CLAUDE.md` (project constitution)

## What I want to build

A CLI tool called rp-stats that scans a directory of Rust projects and
collects per-project statistics. For each project it gathers: project name
from Cargo.toml, whether it's a workspace or single crate, number of crates
if workspace, lines of Rust code, number of source files, number of #[test]
functions, direct dependency count from Cargo.toml, whether a tests/ directory
exists, whether .github/workflows/ exists, and last git commit date and hash.

It writes all results to a JSON registry file. It also generates a single
static HTML file — a dashboard showing each project in a table with summary
stats at the top and basic sorting/filtering on the table columns.

Rust only — it only needs to detect and analyze Rust projects (via Cargo.toml).

## What I need from this session

Brainstorm this design with me using the brainstorming skill. Ask me questions,
explore alternatives, challenge assumptions. When we've converged, write the
output as a `SPEC.md` file with these required sections:

- Problem statement and user persona
- Commands / features (the CLI interface)
- Non-feature list (explicit exclusions with rationale)
- Success checklist (what "done" means — every box checked = v1.0)
- Risk register

Do not write any code in this session. The only output is SPEC.md.