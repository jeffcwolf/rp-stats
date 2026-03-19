# Phase 1 — rp-stats Architecture

## Before anything else

Read these documents in order:
1. `livery/CLAUDE-base.md` — the full constitution
2. `livery/WORKFLOW.md` — Phase 1 section specifically
3. `livery/standards/ousterhout.md` — you will audit the architecture against this
4. `livery/standards/rust-specifics.md` — you will audit API stubs against this
5. `CLAUDE.md` — project constitution
6. `SPEC.md` — the specification produced in Phase 0

If the livery submodule is empty, run `git submodule update --init --recursive`
before reading any livery files.

## What I need from this session

Produce `ARCHITECTURE.md` for rp-stats. No implementation code in this session —
only the architecture document.

Follow the Phase 1 procedure from `livery/WORKFLOW.md` exactly:

1. **Decompose into crates/modules.** Apply the deep-module test from
   `livery/standards/ousterhout.md` to every proposed component: what complexity
   does it hide? If the answer is "not much," collapse it into its parent.
   A small number of deep modules is better than many shallow ones.

2. **Design it twice.** For every non-trivial module interface, sketch two
   fundamentally different approaches. State the tradeoffs. Pick one and
   record why.

3. **Define the public API of each module.** Write real Rust function signatures,
   key types, and error types as stubs. The API is the contract.

4. **Draw the dependency graph.** Every dependency between modules must be
   one-directional and justified. Library crates never depend on the CLI crate.
   Include a Mermaid diagram.

5. **Identify information hiding boundaries.** For each module, list what it
   conceals from callers. If a module exposes its internal representation,
   redesign it.

6. **Define the error handling strategy.** `thiserror` for library errors,
   `anyhow` for the binary crate. Error variants must carry enough context
   to diagnose without reading source.

7. **Run the standards audit.** Work through the Design Process Checklist in
   `livery/standards/ousterhout.md` (Part III) against every module. Then
   apply the Types and Traits checklists from `livery/standards/rust-specifics.md`
   (Part VI) to every public API stub. Fix every violation found. Record
   close calls or intentional tradeoffs as ADR entries.

8. **Record architectural decisions as ADRs.** Numbered entries with context,
   decision, and consequences.

## ARCHITECTURE.md required sections

- Crate map with responsibilities
- Public API stubs for each crate (real Rust signatures)
- Dependency graph (Mermaid diagram)
- Information-hiding inventory per crate
- Error handling strategy
- Workspace Cargo.toml structure (dependency versions)
- Numbered ADR log

## Key constraints from SPEC.md (for reference)

- No external analysis tools (no cargo metadata, no cargo test --list)
- No async
- Git subprocess is the only allowed external tool
- HTML output is a single self-contained static file
- JSON registry is the canonical data format
- Workspace members get individual rows + an aggregated workspace summary row

## Gate condition

Phase 1 is complete when:
- ARCHITECTURE.md exists with all required sections
- The standards audit (step 7) has been completed and all findings resolved
- No Red Flags remain unaddressed
- Every ADR has context, decision, and consequences

Do not proceed to implementation. The only output is ARCHITECTURE.md.