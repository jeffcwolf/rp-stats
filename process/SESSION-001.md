READ  livery/CLAUDE-base.md
READ  livery/WORKFLOW.md
READ  livery/standards/ousterhout.md
READ  livery/standards/rust-specifics.md
READ  CLAUDE.md
READ  SPEC.md
DECISION  Single-crate architecture: rp-stats is small enough (~2k LOC projected) that a workspace with multiple library crates would be shallow decomposition. One binary crate with internal modules provides sufficient information hiding without inter-crate overhead.
DECISION  Module decomposition: scanner (project detection + filesystem walking), collector (stat extraction from files/TOML), registry (JSON serialization), dashboard (HTML generation), git (subprocess interaction). CLI is the thin binary entry point.
DECISION  Design-it-twice for collector: (A) one monolithic collect_stats function vs (B) separate stat-extraction functions composed by the collector module. Chose (B) — each extraction function hides its own parsing complexity and is independently testable.
DECISION  Design-it-twice for dashboard: (A) format! macro string interpolation vs (B) a small HtmlBuilder that appends sections. Chose (A) — the HTML is one file with a known structure; a builder adds a layer that hides no meaningful complexity for a single output shape.
DECISION  Git info via subprocess (`git log -1`): SPEC.md explicitly allows git as the one external tool. Parsing .git internals is fragile (packed refs, shallow clones). Subprocess is simpler and more robust.
ADR  ADR-001: Single binary crate, no workspace — see ARCHITECTURE.md
ADR  ADR-002: Module boundaries — scanner, collector, registry, dashboard, git
ADR  ADR-003: Collector uses composed extraction functions, not monolithic
ADR  ADR-004: HTML generation via format strings, not builder pattern
ADR  ADR-005: Git info via subprocess, not .git parsing
ADR  ADR-006: No traits — all concrete types, no mock need identified
ADR  ADR-007: toml crate for Cargo.toml parsing — SPEC.md forbids custom parser
