# JS/TS Cleanup Patterns

Use these patterns after the main skill has already identified a concrete
problem worth fixing.

## 1. Extraction Test

Extract a helper only if at least one of these is true:

- the new name explains intent better than the old inline block
- the logic is reused or clearly reusable nearby
- the block mixes concerns and separation improves testing or readability

Do not extract if the result is just a named pass-through.

## 2. Parameter Shape

Prefer positional parameters when:

- there are one or two parameters
- order is obvious
- the function is local and stable

Prefer an object parameter when:

- there are several related values
- many call sites only set some fields
- the call site benefits from named arguments

Prefer a typed object in TypeScript when the shape crosses module boundaries.

## 3. Guard Clause Refactor

Use an early return when it removes nesting without scattering the logic.

```ts
function getPayAmount(employee: Employee): PayInfo {
  if (employee.isSeparated) {
    return { amount: 0, reason: 'separated' }
  }

  if (employee.isRetired) {
    return { amount: employee.pension, reason: 'retired' }
  }

  return { amount: employee.salary, reason: 'active' }
}
```

## 4. Validation Boundary

Split these when they are mixed together:

- parsing external input
- business validation
- state mutation or I/O

```ts
async function registerUser(input: unknown) {
  const request = parseRegistration(input)
  validateRegistration(request)

  const user = await createUser(request)
  await sendWelcomeEmail(user)

  return user
}
```

## 5. Error Contract Choice

Follow the local system boundary:

- library-style code often benefits from typed errors or discriminated unions
- app handlers often benefit from framework-native responses
- internal helpers may be fine throwing if callers already expect that pattern

What matters is consistency at the boundary, not one universal rule.

## 6. Types Over Commentary

Prefer:

- narrower function signatures
- explicit unions
- named type aliases for repeated concepts

Over:

- comments explaining the expected shape
- overly generic `any` / `unknown` leakage past the boundary

## 7. Avoid Over-Engineering

Do not replace a simple branch with a class hierarchy unless:

- behavior is truly variant-driven
- the variants are growing independently
- the repo already uses that shape

In many JS/TS codebases, a plain object map or a small switch is cleaner.
