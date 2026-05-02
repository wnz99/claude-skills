---
name: clean-code-py
description: Improve Python readability and maintainability when the task explicitly involves refactoring, code quality, naming, function design, API clarity, or reducing accidental complexity. Use this skill for Python cleanup reviews or targeted refactors. Do not use it to force heavy OOP patterns, override repo-local conventions, or broaden a bug-fix task into a style rewrite.
---

# Clean Code for Python

Inspired by *Clean Code* by Robert C. Martin (Robert Cecil Martin), adapted
for AI-agent use in modern Python codebases.

Use this skill to make Python code more direct, more readable, and easier to
change without losing the Pythonic style of the surrounding codebase.

This skill should help an agent make smaller, better edits, not turn every file
into a lecture on "Clean Code."

## Use This Skill When

- The user explicitly asks for cleaner Python code or a refactor
- A review task should surface maintainability problems, not just bugs
- Naming, function boundaries, data shapes, or error handling are making the code hard to reason about
- A local, behavior-preserving refactor would materially simplify the code

## Do Not Use This Skill When

- The task is mainly to fix behavior and cleanup would noticeably expand scope
- The existing style is already clear and consistent with the project
- The code is intentionally procedural, framework-driven, or generated
- The refactor would introduce unnecessary class hierarchies or abstract base classes just to satisfy style doctrine

## Operating Rules

1. Read local conventions first.
   Check formatter, linter, type-checker, framework, and package-level patterns before applying generic advice.
2. Prefer Pythonic clarity over abstract purity.
   Explicit and readable beats "clever clean."
3. Keep changes local.
   Refactor only the concrete smell you can identify.
4. Preserve contracts.
   Do not silently change return shapes, exception behavior, or mutability expectations unless requested.
5. The surrounding project wins.
   Repo-local patterns override this skill.

## Workflow

### 1. Identify the precise problem

Typical smells:

- unclear names
- one function doing validation, transformation, and side effects together
- unstable or misleading return contract
- repeated ad hoc dict shapes
- deep nesting or rightward drift
- comments compensating for weak structure
- too much incidental object ceremony

### 2. Check local patterns

Before refactoring, inspect nearby files for:

- whether the project prefers plain functions, dataclasses, Pydantic models, or classes
- how it signals expected failures: exceptions, `None`, result-like objects, framework responses
- docstring style and type-hint expectations
- async patterns and resource-management style

### 3. Make the smallest valuable change

Good refactors:

- rename for precision
- split parsing / validation / side effects
- introduce a dataclass or `TypedDict` for a repeated, stable shape
- use keyword-only parameters for call-site clarity
- flatten control flow with guard clauses
- move resource cleanup into a context manager

Avoid:

- creating classes for simple data flow
- introducing inheritance when composition or plain functions are clearer
- extracting helpers that hide the main logic rather than clarifying it
- adding docstrings to every private helper without value

### 4. Verify the runtime contract

Check that:

- callers still receive the expected type or exception behavior
- type hints, tests, and lints still pass
- the refactor reduced cognitive load rather than redistributing it

## Python Heuristics

### Naming

- Prefer domain names over temporary implementation names
- Avoid redundant suffixes such as `data`, `info`, or `object` unless they distinguish real concepts
- Keep terms consistent across modules

### Functions

- A function should have one main responsibility
- Prefer 0-2 positional arguments when possible
- Use keyword-only arguments when they improve call-site readability
- Introduce a `dataclass`, `TypedDict`, or validated model only when the shape is stable and meaningful

### Data Shapes

- Use plain dicts for small local glue code
- Use `TypedDict` for structured mapping-shaped data
- Use `dataclass` for value-like data with behavior-light semantics
- Use richer models only when validation, serialization, or invariants justify them

### Error Handling

- Do not apply "exceptions everywhere" blindly
- Follow the local API style:
  if the codebase raises for expected domain failures, do that cleanly;
  if it uses `None` for "not found" and exceptions for real failure, preserve that distinction
- Wrap low-level exceptions when that improves the boundary seen by callers
- Avoid mixing "not found," "invalid input," and "system failure" into one vague return path

### Control Flow

- Prefer guard clauses when they make the happy path clearer
- Separate input normalization from business logic when they are currently entangled
- Keep async orchestration linear and readable

### Comments and Docstrings

- Comments should explain why, constraints, or surprising behavior
- Docstrings are most useful for public APIs, stable interfaces, and non-obvious behavior
- Do not add commentary that only repeats the code

## Review Checklist

- Is the code easier to scan top to bottom?
- Are names aligned with nearby modules?
- Is the return / exception contract explicit?
- Did the refactor stay Pythonic, or did it introduce unnecessary ceremony?
- Would another maintainer understand the call sites faster now?

## References

Read [references/patterns.md](references/patterns.md) when you need concrete
Python cleanup patterns and examples.
