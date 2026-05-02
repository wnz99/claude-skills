---
name: clean-code-rust
description: Improve Rust readability and maintainability when the task explicitly involves refactoring, code quality, API clarity, ownership design, error handling, or reducing accidental complexity. Use this skill for Rust cleanup reviews or targeted refactors. Do not use it to fight the crate's established patterns, hide useful explicitness, or broaden a bug fix into an unnecessary rewrite.
---

# Clean Code for Rust

Inspired by *Clean Code* by Robert C. Martin (Robert Cecil Martin), adapted
for AI-agent use in idiomatic Rust codebases.

Use this skill to make Rust code more idiomatic, more readable, and safer to
change while respecting the existing crate's conventions and API boundaries.

This skill should help an agent improve code structure without erasing useful
explicitness that Rust intentionally makes visible.

## Use This Skill When

- The user asks for cleaner Rust code, refactoring, or maintainability improvements
- A review task should include design and ergonomics findings, not just correctness bugs
- Ownership, borrowing, error handling, or type design are making the code harder to evolve
- A local refactor can materially simplify the code without changing behavior

## Do Not Use This Skill When

- The task is mainly to fix behavior and a cleanup rewrite would expand scope
- The existing crate already follows a coherent internal style
- The "cleanup" would hide ownership or error behavior that should stay explicit
- The refactor would introduce abstraction layers that are larger than the problem

## Operating Rules

1. Read local crate conventions first.
   Check `Cargo.toml`, formatting, lint settings, error libraries, and nearby module patterns before applying generic advice.
2. Prefer explicit correctness over abstract neatness.
   Rust is allowed to be explicit when that protects invariants.
3. Borrow by default, but not dogmatically.
   Avoid ownership churn when a borrow works; take ownership when the API genuinely needs it.
4. Make the type system do useful work.
   Use types to prevent mistakes when the domain boundary justifies it.
5. Keep refactors local.
   Do not redesign the crate unless the user asked for that.

## Workflow

### 1. Identify the concrete problem

Typical smells:

- unnecessary clones
- weak ownership boundary
- confusing `Result` / `Option` flow
- stringly typed domain concepts
- overgrown functions mixing validation, transformation, and I/O
- panic-based handling in a place that should return errors
- `match` / branching structure that obscures the real state machine

### 2. Check local patterns

Inspect nearby code for:

- whether the crate is app code or library code
- error strategy: `thiserror`, `anyhow`, custom enums, or something else
- public API surface and documentation expectations
- iterator-heavy vs imperative local style
- whether `Arc`, `Rc`, builders, newtypes, or enums are already common patterns

### 3. Pick the smallest meaningful refactor

Good refactors:

- replace an unnecessary owned parameter with a borrow
- remove a clone by restructuring lifetimes or data flow
- split parsing / validation / side effects
- introduce an enum or newtype where raw primitives are too weak
- replace `unwrap` with structured propagation where the boundary requires it
- flatten nested `if let` / `match` when `let-else` or helper methods improve readability

Avoid:

- forcing iterator chains when a loop is clearer
- hiding all branching behind traits prematurely
- introducing builders or wrapper types for one local function call
- replacing explicit, correct Rust with "cleaner" but less obvious control flow

### 4. Verify the boundary

After refactoring, check:

- ownership still matches how callers use the API
- error types still fit the crate boundary
- public behavior and visibility are unchanged unless intended
- `cargo fmt`, `cargo clippy`, and relevant tests still pass

## Rust Heuristics

### Naming and API Shape

- Use domain names, not implementation placeholders
- Avoid stuttering when the module already provides context
- For getters, prefer `name()` over `get_name()` unless `get_` is the local convention or the API semantics justify it

### Ownership

- Prefer `&str` over `String`, `&[T]` over `Vec<T>`, and `&Path` / `impl AsRef<Path>` over `PathBuf` when ownership is unnecessary
- Before using `.clone()`, ask whether the API boundary is wrong or whether shared ownership should be explicit
- Use `Arc` / `Rc` when ownership is genuinely shared, not as a reflex

### Types

- Use enums when states are mutually exclusive
- Use newtypes when raw primitives are easy to mix up
- Do not create wrapper types unless they meaningfully protect the domain or clarify the API

### Error Handling

- In library-like boundaries, prefer `Result` over `panic!` / `unwrap()` for expected failures
- In app code, follow the crate's chosen error surface; `anyhow`-style context may be appropriate
- Keep `Option` for absence and `Result` for failure when the distinction matters
- Do not stringify errors early if callers benefit from structured information

### Control Flow

- Prefer the clearest form, not the shortest one
- Use `let-else`, helper methods, or focused helpers when they flatten noise
- Use iterators when they improve readability; keep loops when they are easier to follow
- Prefer exhaustive `match` on known enums when future variants matter

### Documentation and Comments

- Document public APIs when the crate expects it or the behavior is non-obvious
- `unsafe` blocks should always explain the safety invariant
- Comments should explain why, invariants, or tricky tradeoffs, not narrate obvious code

## Review Checklist

- Is ownership obvious at the API boundary?
- Are there clones that exist only to satisfy the current structure?
- Is the error surface appropriate for app code vs library code?
- Did the refactor improve readability without hiding important state transitions?
- Did the new types earn their weight?

## References

Read [references/patterns.md](references/patterns.md) when you need concrete
Rust cleanup patterns and examples.
