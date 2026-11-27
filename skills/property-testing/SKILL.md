---
name: property-testing
description: Write CsCheck property-based tests that prove every domain rule over random inputs and random sequences of operations. Use whenever a domain guard is added or changed.
---

# Property-based testing

Example tests show that one case works. Properties show that a rule holds for every input the generator can make. Every domain rule gets properties; examples are only for documenting a specific case or replaying a saved failure.

## Tools

- [CsCheck](https://github.com/AnthonyLloyd/CsCheck) with xUnit. Test project: `tests/<App>.Domain.Tests`.
- One test file per aggregate or value object: `<Type>Properties.cs`.

## What to write, for each rule

1. **The invariant holds after any sequence.** Generate a random list of operations (valid and invalid), apply it, and check the invariant after every step. Compare against a tiny model (an `int`, a `List`) that is obviously correct. This is the most important property: it finds bugs no one thought of.
2. **A failed operation changes nothing.** When a guard throws, the object is equal to its state before the call.
3. **Valid input is accepted, invalid input is rejected.** Generate both sides of each boundary: just inside, just outside, far outside.
4. **Round trips.** `Parse(x.ToString()) == x`, `Restore(state) == original`, "receive n then issue n" returns to the start.

Generate edge values on purpose: `0`, `1`, `-1`, `int.MaxValue`, empty strings, the longest allowed string and one more.

```csharp
[Fact]
public void On_hand_never_goes_negative_and_matches_the_model() =>
    Gen.Select(Gen.Bool, Gen.Int[-5, 50]).Array[0, 60].Sample(ops =>
    {
        var item = StockItem.Create(Sku.Parse("SKU-1"));
        var model = 0;
        foreach (var (receive, qty) in ops)
        {
            if (receive && qty > 0) { item.Receive(qty); model += qty; }
            else if (!receive && qty > 0 && qty <= model) { item.Issue(qty); model -= qty; }
            else Assert.ThrowsAny<DomainException>(() => { if (receive) item.Receive(qty); else item.Issue(qty); });
            Assert.True(item.OnHand >= 0);
            Assert.Equal(model, item.OnHand);
        }
    });
```

## Failures and seeds

- When a property fails, CsCheck prints the shrunk input and a seed. Fix the code using the shrunk input.
- Keep the failure: add the seed to the same property (`.Sample(..., seed: "<seed>")`) in a separate `[Fact]` named `Regression_<short description>`. Saved seeds run in every tier.

## Rules for the agent

- **Existing properties are read-only.** Never weaken, delete or skip a property to make it pass. If you believe a property is wrong, stop and report it.
- A property must fail when the rule is removed. Check this with mutation testing (see `mutation-testing`), not by hand.
- Keep the default iteration count in the agent loop (fast). Bigger runs (`iter:` or `CsCheck_Iter` env var) belong to the nightly tier.
