# Sessionizer — Development Guidelines

## Quick Reference

- **Error handling:** `thiserror` (libraries), `color-eyre` (binaries)
- **Logging:** `tracing` + `tracing-subscriber` — structured, span-based
- **Task runner:** `xtask` only — no Makefile, justfile, or scripts/
- **Dev watcher:** `bacon` — see `bacon.toml`
- **Pre-commit hooks:** `cargo xtask lint --install-hooks`
- **Commits:** Conventional Commits — see [commit-and-release.md](./.claude/context/commit-and-release.md)
- **Releases:** `cargo xtask release` — see [commit-and-release.md](./.claude/context/commit-and-release.md)
- **Rust toolchain:** pinned in `rust-toolchain.toml` to match MSRV
- **GitHub CLI:** always use `gh auth switch --user cloudbridgeuy` before any `gh` command

## Unnegotiables

### Crate Boundary FCIS (MUST)

Crate boundaries enforce the Functional Core / Imperative Shell split.

**The rule:** Any crate with `tokio`, AWS SDKs, `reqwest`, or any I/O dependency is an **I/O crate**. I/O crates MUST NOT be depended on by pure crates. If a type in an I/O crate is needed elsewhere, it MUST move down to a pure crate.

- **Pure crates** — types, traits, pure functions. No I/O deps. Any crate can depend on them.
- **I/O crates** — consume pure crate types, add side effects. Depend downward only.
- **Naming** — pure: `sessionizer{domain}_core`. I/O: `sessionizer{domain}` (no `_core` suffix).

### Visibility (MUST)

- `pub(crate)` default for internal functions and types
- `pub` only for public API surface
- No `pub` struct fields — use constructor functions (Parse Don't Validate)

### Error Types (MUST)

Each crate defines `Error` and `Result<T> = std::result::Result<T, Error>`. No domain-prefixed error names (no `AuthnError`). Disambiguate with `sessionizerauthn_core::Error`.

### Clippy (MUST)

- `#![deny(clippy::unwrap_used, clippy::expect_used)]` in every lib.rs and main.rs
- Workspace lints enforce pattern compliance — see [linting-and-clippy.md](./.claude/context/linting-and-clippy.md)
- Test code may use `.unwrap()`

### Verification (MUST)

**`cargo xtask lint` is the single source of truth for code quality.** Run it to validate all changes. Do NOT run `cargo fmt`, `cargo clippy`, `cargo test`, or `cargo check` individually — `xtask lint` runs them all in the correct order and with the correct flags.

- **Before claiming work is done:** run `cargo xtask lint` and confirm exit code 0 (zero output = pass)
- **To auto-fix:** `cargo xtask lint --fix` (applies formatting + clippy fixes)
- Pipeline details: see [xtask-lint.md](./.claude/context/xtask-lint.md)

### Code Quality

- No dead code
- No file over 1000 lines (enforced by xtask) — split at ~300 lines
- `cargo-rail` for dependency unification, dead feature detection, MSRV enforcement
- `cargo-deny` for license and advisory auditing

### Module Organization

Start flat (`src/error.rs`). Promote to directory module when a file exceeds ~300 lines.

### Git Commits (MUST)

Conventional Commits required for `git-cliff`. Full reference: [commit-and-release.md](./.claude/context/commit-and-release.md)

Format: `<type>(<scope>): <description>`. Breaking changes: add `!`. Scopes: crate suffix (e.g., `authn-core`, `sdk`, `cli`).

## Patterns

See `~/.claude/patterns/` for architectural patterns:

- **Functional Core / Imperative Shell** — enforced at crate boundaries
- **Type-Driven Development** — types are the spec; typestate for auth flows
- **Make Impossible States Impossible** — enum variants, not boolean flags
- **Parse Don't Validate** — at system boundaries
- **CQRS** — command/query separation

## Workspace Structure

```
crates/
│  Pure (no I/O)
├── core/              sessionizercore — shared primitives, traits, error types
│  I/O
│  Binaries
├── cli/               sessionizercli — developer CLI (binary: sessionizer)
```

Each crate's `README.md` describes what it owns and its pure/I/O classification.

## Context Documents

| Document                                                           | Purpose                                                                 |
| ------------------------------------------------------------------ | ----------------------------------------------------------------------- |
| [Linting and Clippy](./.claude/context/linting-and-clippy.md)      | Clippy thresholds, workspace lints, and how they map to design patterns |
| [Commit and Release](./.claude/context/commit-and-release.md)      | Conventional commits, version bump logic, release flow                  |
| [xtask lint](./.claude/context/xtask-lint.md)                      | Lint pipeline checks, flags, architecture, adding new checks            |

### Local-Only Documents (MUST NOT commit)

Plans (`.claude/plans/`) and designs (`.claude/designs/`) are **local-only** working documents. They are gitignored and must never be pushed to origin. Only `.claude/context/` and `.claude/commands/` are tracked in git.
