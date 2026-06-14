---
name: tdd
description: Test-driven development with red-green-refactor loop. Use when user wants to build features or fix bugs using TDD, mentions "red-green-refactor", wants integration tests, or asks for test-first development.
---

# Test-Driven Development

## Philosophy

**Core principle**: Tests should verify behavior through public interfaces, not implementation details. Code can change entirely; tests shouldn't.

**Good tests** are integration-style: they exercise real code paths through public APIs. They describe _what_ the system does, not _how_ it does it. A good test reads like a specification — "user can checkout with valid cart" tells you exactly what capability exists. These tests survive refactors because they don't care about internal structure.

**Bad tests** are coupled to implementation. They mock internal collaborators, test private methods, or verify through external means (like querying a database directly instead of using the interface). The warning sign: your test breaks when you refactor, but behavior hasn't changed.

## Language Conventions

Before writing any test, identify the language and apply its idiomatic tooling:

| Language   | Default test runner     | Assertion style            | Mock/stub boundary tool         |
|------------|-------------------------|----------------------------|---------------------------------|
| TypeScript | Vitest / Jest           | `expect(x).toBe(y)`        | `vi.fn()` / `jest.fn()`         |
| Python     | pytest                  | plain `assert`             | `unittest.mock.patch`           |
| Java       | JUnit 5 + AssertJ       | `assertThat(x).isEqualTo(y)` | Mockito (boundaries only)     |
| Go         | `testing` + `testify`   | `assert.Equal(t, ...)`     | interface substitution          |
| Rust       | built-in `#[test]`      | `assert_eq!` / `assert!`   | trait substitution              |
| Ruby       | RSpec                   | `expect(x).to eq(y)`       | `instance_double`               |
| C#         | xUnit + FluentAssertions| `x.Should().Be(y)`         | Moq (boundaries only)           |

If the project already has a test runner in use, match it — don't introduce a second framework.

Follow the language's naming convention for test files and test functions:
- Python: `test_*.py`, functions named `test_*`
- Go: `*_test.go`, functions named `TestXxx`
- Java: `*Test.java`, methods named `testXxx` or descriptive with `@Test`
- TypeScript/JS: `*.test.ts` or `*.spec.ts`
- Rust: `#[cfg(test)]` module at the bottom of the file or in `tests/`
- Ruby: `*_spec.rb`
- C#: `*Tests.cs`

## Anti-Pattern: Horizontal Slices

**DO NOT write all tests first, then all implementation.** This is "horizontal slicing" — treating RED as "write all tests" and GREEN as "write all code."

This produces bad tests:
- Written in bulk against imagined behavior, not actual behavior
- Test the shape of things (data structures, signatures) rather than user-facing behavior
- Insensitive to real changes — pass when behavior breaks, fail when behavior is fine
- You outrun your headlights, committing to test structure before understanding the implementation

**Correct approach**: Vertical slices via tracer bullets. One test → one implementation → repeat.

```
WRONG (horizontal):
  RED:   test1, test2, test3, test4, test5
  GREEN: impl1, impl2, impl3, impl4, impl5

RIGHT (vertical):
  RED→GREEN: test1→impl1
  RED→GREEN: test2→impl2
  RED→GREEN: test3→impl3
  ...
```

## Workflow

### 1. Planning

Before writing any code:

- [ ] Identify the language and its idiomatic test tooling (see Language Conventions above)
- [ ] Confirm with user what interface changes are needed
- [ ] Confirm with user which behaviors to test (prioritize)
- [ ] Identify opportunities for deep modules (small interface, deep implementation)
- [ ] Design interfaces for testability
- [ ] List the behaviors to test — not implementation steps
- [ ] Get user approval on the plan

Ask: "What should the public interface look like? Which behaviors are most important to test?"

**You can't test everything.** Focus on critical paths and complex logic, not every edge case.

### 2. Tracer Bullet

Write ONE test that confirms ONE thing about the system:

```
RED:   Write test for first behavior → test fails
GREEN: Write minimal code to pass → test passes
```

This is your tracer bullet — proves the path works end-to-end.

### 3. Incremental Loop

For each remaining behavior:

```
RED:   Write next test → fails
GREEN: Minimal code to pass → passes
```

Rules:
- One test at a time
- Only enough code to pass the current test
- Don't anticipate future tests
- Keep tests focused on observable behavior

### 4. Refactor

After all tests pass, look for refactor candidates:

- [ ] Extract duplication
- [ ] Deepen modules (move complexity behind simple interfaces)
- [ ] Apply language-appropriate design principles (SOLID, traits, typeclasses, protocols)
- [ ] Consider what new code reveals about existing code
- [ ] Run tests after each refactor step

**Never refactor while RED.** Get to GREEN first.

## Checklist Per Cycle

```
[ ] Test describes behavior, not implementation
[ ] Test uses public interface only
[ ] Test would survive internal refactor
[ ] Code is minimal for this test
[ ] No speculative features added
[ ] Test follows language naming and structure conventions
```

---

# Good and Bad Tests

## Good Tests

**Integration-style**: test through real interfaces, not mocks of internal parts.

The same principle in three languages:

```python
# Python — GOOD: tests observable behavior
def test_user_can_checkout_with_valid_cart():
    cart = create_cart()
    cart.add(product)
    result = checkout(cart, payment_method)
    assert result.status == "confirmed"
```

```java
// Java — GOOD: tests observable behavior
@Test
void userCanCheckoutWithValidCart() {
    Cart cart = createCart();
    cart.add(product);
    CheckoutResult result = checkout(cart, paymentMethod);
    assertThat(result.getStatus()).isEqualTo("confirmed");
}
```

```go
// Go — GOOD: tests observable behavior
func TestUserCanCheckoutWithValidCart(t *testing.T) {
    cart := createCart()
    cart.Add(product)
    result, _ := checkout(cart, paymentMethod)
    assert.Equal(t, "confirmed", result.Status)
}
```

Characteristics (language-agnostic):
- Tests behavior callers care about
- Uses public API only
- Survives internal refactors
- Describes WHAT, not HOW
- One logical assertion per test

## Bad Tests

**Implementation-detail tests**: coupled to internal structure.

```python
# Python — BAD: asserts on internal call, not behavior
def test_checkout_calls_payment_service(mocker):
    mock_payment = mocker.patch("app.payment_service.process")
    checkout(cart, payment)
    mock_payment.assert_called_once_with(cart.total)
```

```java
// Java — BAD: bypasses interface to verify
@Test
void createUserSavesToDatabase() throws Exception {
    createUser(new User("Alice"));
    User row = jdbcTemplate.queryForObject("SELECT * FROM users WHERE name = ?", ...);
    assertThat(row).isNotNull();
}

// Java — GOOD: verifies through interface
@Test
void createUserMakesUserRetrievable() {
    User created = createUser(new User("Alice"));
    User retrieved = getUser(created.getId());
    assertThat(retrieved.getName()).isEqualTo("Alice");
}
```

Red flags (language-agnostic):
- Mocking internal collaborators
- Testing private/package-private methods
- Asserting on call counts or call order
- Test breaks on refactor with no behavior change
- Test name describes HOW not WHAT
- Bypassing the public interface to verify state

---

# When to Mock

Mock at **system boundaries** only:

- External APIs (payment, email, SMS, etc.)
- Databases — prefer a real test DB; mock only when setup cost is prohibitive
- Time and randomness
- File system — prefer temp directories; mock only when impractical

**Do not mock** your own modules, internal collaborators, or anything you control.

## Designing for Mockability

The same dependency injection principle applies in every language — pass boundaries in, don't create them internally:

```python
# Python — injectable boundary
def process_payment(order, payment_client):
    return payment_client.charge(order.total)

# Python — hard to test
def process_payment(order):
    client = StripeClient(os.environ["STRIPE_KEY"])
    return client.charge(order.total)
```

```go
// Go — use an interface for the boundary
type PaymentClient interface {
    Charge(amount int) (Receipt, error)
}

func ProcessPayment(order Order, client PaymentClient) (Receipt, error) {
    return client.Charge(order.Total)
}
// In tests, pass a fake struct that satisfies the interface — no mocking library needed.
```

```java
// Java — inject via constructor
class OrderService {
    private final PaymentGateway gateway;
    OrderService(PaymentGateway gateway) { this.gateway = gateway; }
    Receipt processPayment(Order order) { return gateway.charge(order.getTotal()); }
}
// In tests, pass a Mockito mock or a hand-written fake.
```

**Prefer SDK-style interfaces over generic fetchers** — one function per external operation, not one generic dispatcher. Each function is independently substitutable and its shape is unambiguous in tests.

---

# Interface Design for Testability

These properties make interfaces testable regardless of language:

1. **Accept dependencies, don't create them** (dependency injection)
2. **Return results, don't produce hidden side effects**
3. **Small surface area** — fewer methods means fewer tests needed, simpler setup

```python
# GOOD: returns result, accepts dependency
def calculate_discount(cart, pricing_rules) -> Discount: ...

# BAD: mutates in place, creates own dependency
def apply_discount(cart) -> None:
    rules = load_rules_from_db()
    cart.total -= rules.compute(cart)
```

In typed languages (Java, Go, Rust, C#, TypeScript), model boundaries as interfaces or traits — not concrete classes. This makes substitution in tests natural without any mocking library.

---

# Deep Modules

From *A Philosophy of Software Design*:

**Deep module** = small interface + lots of implementation

```
┌─────────────────────┐
│   Small Interface   │  ← few methods, simple params
├─────────────────────┤
│                     │
│  Deep Implementation│  ← complex logic hidden inside
│                     │
└─────────────────────┘
```

**Shallow module** = large interface + little implementation (avoid)

```
┌─────────────────────────────────┐
│       Large Interface           │  ← many methods, complex params
├─────────────────────────────────┤
│  Thin Implementation            │  ← just passes through
└─────────────────────────────────┘
```

When designing a module, ask:
- Can I reduce the number of methods?
- Can I simplify the parameters?
- Can I hide more complexity inside?

Deep modules are easier to test because there are fewer entry points and the interface rarely changes.

---

# Refactor Candidates

After the TDD cycle, look for:

- **Duplication** → extract function, method, or module
- **Long functions** → break into private helpers (keep tests on the public interface)
- **Shallow modules** → combine or deepen
- **Feature envy** → move logic to where data lives
- **Primitive obsession** → introduce value objects or newtypes
- **Existing code** that the new code reveals as problematic

Language-specific patterns to consider during refactor:
- Python: dataclasses, `__slots__`, protocol classes for structural typing
- Go: embedding, interface composition
- Java: records, sealed classes, streams for collection pipelines
- TypeScript: discriminated unions, `readonly`, `satisfies`
- Rust: enums with data, `impl Trait`, the newtype pattern
- Ruby: modules for mixins, `Comparable`, `Enumerable`
- C#: records, extension methods, `IEnumerable` pipelines

---

# TDD Pipeline

This section is machine-readable and is invoked by `ralph` via the `Skill` tool when `mode: "pipeline"` is set.

---

## Input Contract

When `ralph` invokes `tdd` via `Skill` tool in pipeline mode, it passes:

```json
{
  "task_file": ".seed/{number}/tasks/NNN-*.md",
  "acceptance_criteria": ["Criterion 1", "Criterion 2"],
  "language": "TypeScript",
  "mode": "pipeline"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `task_file` | string | Path to the task file produced by `issues` |
| `acceptance_criteria` | string[] | List of acceptance criteria for this slice |
| `language` | string | Programming language for the project |
| `mode` | `"pipeline"` | Must be `"pipeline"` to activate pipeline mode |

---

## Output Contract

`tdd` in pipeline mode returns structured output:

```json
{
  "status": "SUCCESS | PARTIAL | FAILED",
  "stage": "tdd",
  "test_files_written": ["path/to/test1.ts", "path/to/test2.ts"],
  "test_count": 5,
  "tests_passed": true,
  "timestamps": {
    "test_written_at": "ISO8601",
    "implementation_started_at": "ISO8601",
    "all_tests_passing_at": "ISO8601"
  },
  "red_green_refactor_order": true,
  "hitl_needed": false,
  "hitl_reason": null,
  "warnings": [],
  "confidence": 0.9
}
```

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | SUCCESS, PARTIAL, or FAILED |
| `stage` | string | Always "tdd" |
| `test_files_written` | string[] | Paths to test files created |
| `test_count` | number | Number of tests written |
| `tests_passed` | boolean | Whether all tests pass |
| `timestamps.test_written_at` | string | ISO8601 timestamp when first test was written |
| `timestamps.implementation_started_at` | string | ISO8601 timestamp when implementation began |
| `timestamps.all_tests_passing_at` | string | ISO8601 timestamp when all tests passed |
| `red_green_refactor_order` | boolean | True if tests were written before implementation |
| `hitl_needed` | boolean | True if a human decision is required |
| `hitl_reason` | string | Description of the decision needed |
| `warnings` | string[] | Any concerns or caveats |
| `confidence` | number | Confidence score between 0 and 1 |

---

## Pipeline Mode

When `mode: "pipeline"` is set, `tdd` operates autonomously without user approval prompts:

- Suppresses all user approval prompts
- Runs the red-green-refactor loop autonomously
- Emits `hitl_needed: true` if a design decision is required mid-loop
- Returns `status: PARTIAL` if a decision cannot be reached
- Continues until all tests pass or HITL signal is emitted

### HITL Signal

If `tdd` encounters a design decision it cannot make, it emits:
```json
{
  "hitl_needed": true,
  "hitl_reason": "Description of the decision needed"
}
```

`ralph` pauses the TDD loop, surfaces the decision to the user, and resumes after the decision is made.

---

## Status Block

After completing the TDD cycle, emit a trailing Status Block:

```json
{
  "status": "SUCCESS | PARTIAL | FAILED",
  "stage": "tdd",
  "summary": "One-line description of what was accomplished",
  "warnings": ["Any caveats or concerns"],
  "confidence": 0.0-1.0
}
```