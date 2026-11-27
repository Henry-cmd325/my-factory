---
name: domain-modeling
description: Turn plain-language requirements into C# domain classes whose guards enforce every business rule. Use before writing any slice, and whenever a requirement adds or changes a rule.
---

# Domain modeling

The C# domain classes are the documentation of the system. There is no separate spec. A reader must be able to learn every business rule by reading the domain project, so every rule lives there as code, and nowhere else.

## Rules

1. **One rule, one guard.** Each business rule from the request becomes a guard inside the aggregate or value object that owns it. Write the rule in one plain-English `<summary>` line above the guard's member, in the user's words (for example `/// Stock on hand can never go negative.`).
2. **Aggregates protect themselves.** State changes only through methods with business names (`Receive`, `Issue`). No public setters, no public constructors that skip the guards. Use static factories: `Create(...)` for new objects and `Restore(...)` for loading from storage. `Restore` checks the same invariants as `Create`.
3. **Value objects for constrained values.** If a value has a format or a range (a SKU, a quantity), make it a `readonly record struct` with a `Parse` or `From` factory that throws on bad input. A value object never exists in an invalid state. Aggregate methods take value objects, not raw `int` or `string` (`Issue(Quantity quantity)`), so callers must validate input before they load anything.
4. **Fail with domain exceptions, with useful messages.** The message is part of the API (it becomes the ProblemDetails `detail`): name the rule and the rejected value, and test it. Invalid operations throw a type derived from `DomainException`. Use a specific subtype when the API must answer differently (for example `InsufficientStockException` becomes 409, a plain `DomainException` becomes 400). Leave the object unchanged when a guard fails.
5. **Pure and deterministic.** The domain project references only the BCL. No ASP.NET, no database driver, no logging. Pass time and IDs in as arguments; never call `DateTime.Now`, `Guid.NewGuid()` or `Random` inside the domain.
6. **Guard arithmetic.** Check for overflow before adding or multiplying counters. Put explicit upper limits on inputs that come from users.
7. **AOT-safe.** No reflection. Use `[GeneratedRegex]` for patterns.
8. **Persistence details stay out.** Concurrency tokens, row versions and table names belong to the store, not the domain class.

## Layout

```
src/<App>.Domain/
  DomainException.cs
  <Area>/
    <Aggregate>.cs
    <ValueObject>.cs
    <SpecificException>.cs
```

`<Area>` is a business area (`Stock`, `Orders`), never a technical layer.

## Checks that prove it

- Every guard has at least one property in `<App>.Domain.Tests` (see the `property-testing` skill).
- An architecture test fails if `<App>.Domain` references any assembly outside the BCL.
- The mutation score of `<App>.Domain` is reported on every run (see the `mutation-testing` skill). A surviving mutant in a guard means a rule is not really tested.

## Never

- Business rules in endpoints, handlers or SQL only. (A database constraint may repeat a rule as defense in depth, but the domain guard is the source of truth.)
- Anemic classes with public setters plus a "validator" somewhere else.
- `Service` or `Manager` classes in the domain.
