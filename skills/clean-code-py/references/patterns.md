# Python Cleanup Patterns

Use these patterns when the main skill has already identified a specific
maintainability problem.

## 1. Split Responsibilities, Not Sentences

Extract a helper when the function currently mixes:

- input parsing
- validation
- domain logic
- persistence or I/O

```python
async def register_user(payload: dict) -> User:
    request = parse_registration(payload)
    validate_registration(request)
    user = await create_user(request)
    await send_welcome_email(user)
    return user
```

## 2. Choose the Right Data Shape

Use:

- plain `dict` for short-lived local glue
- `TypedDict` for repeated mapping-shaped data
- `dataclass` for value objects with stable fields
- a heavier model only when validation and serialization justify it

Do not upgrade every dict into a class by default.

## 3. Keyword-Only Arguments

Use keyword-only args when clarity at the call site matters more than brevity.

```python
def create_user(
    *,
    email: str,
    role: str,
    is_active: bool = True,
) -> User:
    ...
```

Avoid this when the function is tiny and local and positional arguments are already obvious.

## 4. Guard Clause Refactor

```python
def get_pay_amount(employee: Employee) -> PayInfo:
    if employee.is_separated:
        return PayInfo(amount=0, reason="separated")

    if employee.is_retired:
        return PayInfo(amount=employee.pension, reason="retired")

    return PayInfo(amount=employee.salary, reason="active")
```

Use this when it makes the happy path clearer. Do not flatten code so aggressively that related branching becomes harder to follow.

## 5. Resource Boundaries

If cleanup is manual or repeated, prefer a context manager.

```python
@contextmanager
def database_transaction():
    conn = get_connection()
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()
```

## 6. Error Boundary Rule

Keep these distinct if the project does:

- invalid input
- missing data
- infrastructure failure

Do not collapse them into one `ValueError` or one `None` path unless that is already the project convention.

## 7. Avoid Ceremony

Do not force:

- abstract base classes where a protocol or plain callable works
- inheritance where composition is simpler
- docstrings on every trivial helper
- class wrappers around pure transformations

In Python, directness is usually a feature.
