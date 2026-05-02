# Rust Cleanup Patterns

Use these patterns when the main skill has already identified a real design or
readability problem worth fixing.

## 1. Borrow vs Own

Prefer borrowing when the callee only reads the data.

```rust
fn greet(name: &str) {
    println!("Hello, {name}");
}
```

Take ownership when the function stores, transforms into a new owned value, or
must outlive the caller's borrow.

## 2. Clone Check

Before `.clone()`, ask:

1. does the callee actually need ownership?
2. can I restructure the order of operations?
3. should shared ownership be explicit via `Arc` / `Rc`?

If the answer to all three is no, the clone may still be fine. The goal is not
"zero clones"; the goal is intentional clones.

## 3. Newtype or Raw Primitive

Introduce a newtype when the same primitive is used for multiple domain values
that are easy to swap.

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct UserId(u64);

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Cents(u64);
```

Keep raw primitives when the extra type adds ceremony without protecting anything meaningful.

## 4. Error Boundary Choice

Use the crate boundary as the guide:

- library-facing boundary: typed `Result` with actionable variants
- app-facing orchestration: contextual errors may be enough
- impossible state / invariant: `expect` may be acceptable if documented

Do not use `unwrap()` as a substitute for choosing an error policy.

## 5. `Option` vs `Result`

Use:

- `Option<T>` when absence is normal and not diagnostic
- `Result<T, E>` when the caller needs to know why it failed

Do not collapse both cases into one if the distinction matters to callers.

## 6. Flatten Branching Carefully

Use `let-else` or focused helpers when they make the happy path clearer.

```rust
fn require_admin_email(request: &Request) -> Result<String, AuthError> {
    let Some(user_id) = request.user_id() else {
        return Err(AuthError::Unauthenticated);
    };

    let Some(user) = db::find_user(user_id) else {
        return Err(AuthError::UserGone);
    };

    if !user.is_admin {
        return Err(AuthError::Forbidden);
    }

    user.email.ok_or(AuthError::MissingEmail)
}
```

Do not flatten so aggressively that the state transitions become harder to see.

## 7. Iterator vs Loop

Prefer iterators when they make intent obvious.

Prefer a loop when:

- there is branching or mutation that is clearer step by step
- the iterator chain becomes hard to scan
- error handling or short-circuiting is easier to express imperatively

Readability decides this, not ideology.
