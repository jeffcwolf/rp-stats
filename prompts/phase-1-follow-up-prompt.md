# Phase 1 Follow-Up — Complete ARCHITECTURE.md

## Before anything else

1. If `livery/` is empty, run: `git submodule update --init --recursive`
2. Read `livery/CLAUDE-base.md`
3. Read `livery/standards/ousterhout.md` — Part III (Design Process Checklist)
4. Read `livery/standards/rust-specifics.md` — Part VI (Types and Traits checklists)
5. Read `CLAUDE.md`
6. Read `SPEC.md`
7. Read `ARCHITECTURE.md` — this is the file you are completing

## Context

The initial Phase 1 session produced ARCHITECTURE.md with good module
decomposition, information hiding boundaries, and design decisions. But
several required sections are missing or incomplete, likely due to session
timeouts. This session completes them. No implementation code — only
ARCHITECTURE.md updates.

## What needs to be added

### 1. Public API stubs with real Rust signatures

For every module, add the actual Rust function signatures, struct
definitions, and error types. These are the contracts that implementation
sessions will code against. Example of the level of detail expected:

```rust
// discovery.rs
pub(crate) struct DiscoveredProject {
    pub path: PathBuf,
    pub is_workspace: bool,
}

pub(crate) fn discover(target_dir: &Path, recursive: bool) -> Result<Vec<DiscoveredProject>>;
```

Do this for every module: cli.rs, discovery.rs, collector.rs,
cargo_toml.rs, line_count.rs, test_count.rs, git_info.rs, registry.rs,
html.rs. Include the key structs (Config, DiscoveredProject, ProjectEntry,
MemberEntry, Registry) with their fields.

### 2. Mermaid dependency graph

Add a Mermaid diagram showing which modules depend on which. For a single
crate this is simple but it documents the import relationships and
confirms there are no unexpected couplings.

### 3. Standards audit

Run the Design Process Checklist from `livery/standards/ousterhout.md`
(Part III) against every module in the architecture. Then run the Types
and Traits checklists from `livery/standards/rust-specifics.md` (Part VI)
against every public API stub you just wrote. Record the results:
- Which checks passed
- Any findings and how they were resolved
- Any close calls recorded as ADR entries

### 4. ADR format

Rewrite the Key Design Decisions section as properly formatted ADRs:

```markdown
### ADR-001: Title

**Context:** What situation prompted this decision?

**Decision:** What was decided?

**Consequences:** What are the results — positive and negative?

**Source:** Phase 1 architecture session
```

## Gate condition

This session is complete when ARCHITECTURE.md contains:
- Real Rust function signatures for every module
- Struct definitions with fields for all key types
- A Mermaid dependency graph
- A recorded standards audit with findings
- ADRs in the proper format (Context / Decision / Consequences)

Do not write any implementation code. The only output is the updated
ARCHITECTURE.md.