# Clean Code: Good Practices & Principles

A practical, language-agnostic guide you can keep beside your keyboard. Examples reference **Kotlin/Java** and **Python**, but principles apply broadly.

---

## Table of Contents
1. [Core Principles](#core-principles)
2. [Naming](#naming)
3. [Functions & Methods](#functions--methods)
4. [Comments & Documentation](#comments--documentation)
5. [Formatting & Style](#formatting--style)
6. [Error Handling](#error-handling)
7. [Logging](#logging)
8. [Data Modeling](#data-modeling)
9. [Dependencies & Boundaries](#dependencies--boundaries)
10. [State, Concurrency & Async](#state-concurrency--async)
11. [Public APIs & Services](#public-apis--services)
12. [Testing](#testing)
13. [Refactoring Workflow](#refactoring-workflow)
14. [Performance](#performance)
15. [Security Basics](#security-basics)
16. [Version Control & Reviews](#version-control--reviews)
17. [Definition of Done](#definition-of-done)
18. [Language-Specific Notes](#language-specific-notes)
19. [One-Page Checklist](#one-page-checklist)

---

## Core Principles

- **KISS (Keep It Simple, Straightforward):** Prefer simplest design that works.
- **SRP (Single Responsibility Principle):** One reason to change per module/class/function.
- **Open/Closed:** Open for extension, closed for modification (use composition, strategy).
- **Liskov Substitution:** Subtypes should be usable anywhere their base type is expected.
- **Interface Segregation:** Smaller, focused interfaces > fat ones.
- **Dependency Inversion:** Depend on abstractions, not concretions.
- **DRY (Don’t Repeat Yourself):** Extract duplication into functions/types.
- **YAGNI (You Aren’t Gonna Need It):** Build only what’s needed now, not hypotheticals.
- **Law of Demeter:** Talk to friends, not strangers (avoid long chains like `a.b().c().d()`).
- **Single Level of Abstraction:** Keep code in a scope at one consistent abstraction level.

---

## Naming

Good names compress intent.

- **Be precise & domain-focused:** `CartPriceCalculator` > `Helper`.
- **Avoid encodings & abbreviations:** `calculateSubtotal()` > `calcSubTot()`.
- **Boolean names as predicates:** `isEligible`, `hasAccess`, `shouldRetry`.
- **Collections pluralized:** `orders`, `customerIds`.
- **Constants & enums read like English.**
- **Avoid misleading temporal words:** prefer `requestedAtUtc` over `time`.

**Do**

```kotlin
val allowedDomains: Set<String>
fun isExpired(at: Instant): Boolean
```

**Avoid**

```kotlin
val list1: MutableList<String>
fun check(whenX: Instant): Boolean
```

---

## Functions & Methods

- **Small:** Aim for 5–20 lines. Extract aggressively.
- **One job:** If you need “and” in the description, split it.
- **Few parameters:** 0–3 ideally. Group related ones into a value object.
- **No boolean flags:** Prefer two functions or strategy objects.
- **Minimize side effects:** Prefer pure functions; isolate side effects at boundaries.
- **Return early:** Guard clauses > deep nesting.
- **Fail fast:** Validate inputs and preconditions.
- **Command–Query Separation:** Queries don’t change state; commands don’t return data (beyond status).

**Example (Kotlin – guard clauses and parameter object)**

```kotlin
data class CheckoutContext(val cartId: String, val accountId: String, val currency: String)

fun calculateTotals(ctx: CheckoutContext, items: List<Item>): Totals {
    require(ctx.currency.isNotBlank()) { "currency required" }
    if (items.isEmpty()) return Totals.ZERO
    // ... computations
    return totals
}
```

---

## Comments & Documentation

- **Prefer self-explanatory code** over comments.
- **Comment “why”, not “what”.** Document trade-offs, invariants, non-obvious constraints.
- **API docs:** Public APIs deserve clear docstrings (KDoc/Javadoc, Python docstrings).
- **Keep comments close to code** and update during refactors.

**Good**

```python
# Business rule: free shipping on tool orders over $100 before tax.
```

**Avoid**

```python
# Add 100 to x and compare  # (what, not why)
```

---

## Formatting & Style

- **Automate style:** Use formatters/linters (ktlint/spotless, Black, ruff, ESLint).
- **Consistent wrapping & imports.**
- **Limit line length** (100–120) and **prefer vertical density** with meaningful whitespace.
- **Immutability by default:** `val` over `var`, `final` fields in Java.

---

## Error Handling

- **Use exceptions (or Result types) for exceptional paths.**
- **Don’t swallow exceptions:** add context, rethrow or handle.
- **Wrap low-level errors:** Preserve cause; add domain context.
- **Prefer typed errors/Result** in boundaries where exceptions are noisy.
- **Be specific:** Catch narrow exception types.
- **Maintain failure atomicity:** Partial writes should roll back or be clearly signaled.
- **Validate inputs early; enforce invariants in constructors/factories.**

**Kotlin example (Result)**

```kotlin
sealed interface PaymentResult {
    data class Success(val id: String): PaymentResult
    data class Declined(val reason: String): PaymentResult
    data class Error(val cause: Throwable): PaymentResult
}
```

---

## Logging

- **Structured logs:** key=value pairs, JSON in services.
- **Right level:** `DEBUG` (dev details), `INFO` (state changes), `WARN` (recoverable issues), `ERROR` (user-visible failure).
- **No secrets/PII:** Mask tokens, emails, card data.
- **Correlation IDs & request ids** for tracing.
- **Use parameterized logging** rather than string concatenation.

```kotlin
logger.info("cart_checked_out accountId={} cartId={} total={}", accountId, cartId, total)
```

---

## Data Modeling

- **Prefer value objects** (e.g., `Email`, `Money`, `Percent`) over primitives.
- **Make illegal states unrepresentable:** nullable only when truly optional.
- **Use enums/sealed classes** for closed sets of states.
- **Avoid nulls:** use `Optional` (Java) or Kotlin null-safety; provide safe defaults where valid.
- **Time & currency:** store in canonical forms (UTC timestamps, ISO currency).

```kotlin
@JvmInline value class Money(val cents: Long)
```

---

## Dependencies & Boundaries

- **Dependency Injection:** constructor injection for testability.
- **Bounded contexts:** keep domain logic separate from transport (HTTP, Kafka).
- **Ports & Adapters (Hexagonal):** domain depends on interfaces; adapters implement them.
- **Feature toggles** for incremental delivery; remove stale toggles.

```kotlin
interface PaymentPort { fun charge(amount: Money): PaymentId }
class StripeAdapter(...): PaymentPort { ... }
```

---

## State, Concurrency & Async

- **Minimize shared mutable state;** prefer immutability.
- **Use high-level primitives:** executors, coroutines/flows, futures, channels.
- **Set timeouts & retries with jitter;** avoid unbounded backoff.
- **Idempotency:** make operations safe to retry.
- **Cancellation-aware** async code.

```kotlin
withTimeout(2_000) { service.call() }
```

---

## Public APIs & Services

- **Predictable resource modeling** and **consistent naming**.
- **HTTP semantics:** use correct verbs and status codes.
- **Idempotency keys** for POST where needed.
- **Pagination:** `limit`, `cursor/next`, Link headers.
- **Errors:** Use Problem Details (`application/problem+json`) with `type`, `title`, `status`, `detail`, `instance`.
- **Versioning:** URI or content negotiation; prefer backward-compatible changes.
- **Security:** authn/authz, rate limits, input validation, JSON schema.

---

## Testing

- **Test pyramid:** many unit tests, fewer integration, few e2e.
- **AAA pattern:** Arrange–Act–Assert; one behavior per test.
- **Readable names:** backtick or sentence-style names.
- **Deterministic:** no sleeps/time-based flakiness; use fakes and time abstractions.
- **Contract tests** for service and message boundaries.
- **Avoid over-mocking:** mock only external collaborators.
- **Property-based tests** for critical pure logic.
- **Fixtures/builders:** remove duplication.

```kotlin
@Test fun `subtotal = sum(price * qty)`() { /* AAA */ }
```

---

## Refactoring Workflow

- **Small, safe steps:** keep code running.
- **Cover with tests first** (or add characterization tests for legacy).
- **Common moves:** Extract Function, Introduce Parameter Object, Replace Conditional with Polymorphism, Invert Dependencies, Strangler Fig for legacy systems.
- **Boy Scout Rule:** leave code a little cleaner than you found it.

---

## Performance

- **Measure before optimizing:** use profilers/metrics.
- **Watch allocations & hot paths;** prefer streaming over loading everything.
- **Beware N+1 queries;** batch or prefetch.
- **Cache with discipline:** invalidation strategy, TTLs, size bounds.
- **Backpressure** in async streams.

---

## Security Basics

- **Validate & sanitize inputs** (server-side always).
- **Least privilege** for services and DB users.
- **Secrets management:** never hardcode; rotate regularly.
- **Avoid detailed error leaks** to clients; log securely.
- **HTTPS everywhere;** HSTS, secure cookies, CSRF where relevant.
- **OWASP Top 10** awareness: injection, broken auth, sensitive data exposure, etc.

---

## Version Control & Reviews

- **Small, focused commits** with meaningful messages.
- **Descriptive PRs:** context, screenshots, rollout plan.
- **Automate checks:** tests, linters, formatters, SAST.
- **Review for correctness, clarity, and consistency,** not just style.
- **Be kind & specific** in feedback; propose concrete improvements.

---

## Definition of Done

- [ ] Functionally complete & acceptance criteria met  
- [ ] Code formatted, linted, and self-explanatory  
- [ ] Tests written/updated (unit/integration/contract) and passing  
- [ ] Logging/metrics/tracing added where useful  
- [ ] Errors mapped to client-friendly responses  
- [ ] Security/privacy reviewed; secrets externalized  
- [ ] Backward compatibility considered (migrations, API changes)  
- [ ] Docs/READMEs updated; runbooks/playbooks updated if needed  
- [ ] Deployment/rollback plan prepared; feature flags gated if applicable  

---

## Language-Specific Notes

### Kotlin/Java
- Prefer `val`/`final` and immutable collections.
- Use data classes & value classes for domain types.
- Leverage sealed classes for result/state modeling.
- Avoid `@Nullable` soup; design with null-safety.
- Parameterized logging; avoid string concatenation.
- Use DI frameworks (Spring, Koin, Dagger) responsibly (constructor injection).

### Python
- Follow PEP 8; type hints with `mypy` for critical code.
- Prefer pathlib, dataclasses/pydantic for models.
- Use context managers; avoid bare `except:`.
- Virtualenv/uv/poetry for reproducible envs.
- Black + ruff to format/lint automatically.

### JavaScript/TypeScript
- Favor TypeScript for safety.
- Use ESLint + Prettier; strict compiler options.
- Avoid implicit `any`; model domain types precisely.
- Keep side effects out of reducers/selectors.

---

## One-Page Checklist

- [ ] Names express domain intent  
- [ ] Functions are small, do one thing, few params, no boolean flags  
- [ ] Guard clauses > deep nesting  
- [ ] Errors handled with context; no swallowing  
- [ ] Structured logging w/ correlation IDs; no secrets  
- [ ] Value objects make illegal states unrepresentable  
- [ ] Dependencies inverted; boundaries well-defined  
- [ ] Concurrency safe; timeouts/retries/cancellation in place  
- [ ] Public APIs consistent; proper HTTP semantics & error format  
- [ ] Tests are fast, deterministic, readable; avoid over-mocking  
- [ ] Refactors done in small steps; code left cleaner  
- [ ] Performance measured, not guessed; no premature optimization  
- [ ] Security basics applied; secrets managed  
- [ ] PRs small and well-described; CI green; docs updated

---

**Tip:** Pin this file in your repo/wiki and update it as your team’s standards evolve.
