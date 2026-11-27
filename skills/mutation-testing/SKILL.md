---
name: mutation-testing
description: Run Stryker.NET to check that tests really catch bugs, read the report, and kill surviving mutants with better tests. Use after adding or changing domain code or its tests, and on every pull request.
---

# Mutation testing

Stryker.NET changes the code on purpose (flips `>=` to `>`, removes a statement, negates a condition) and runs the tests. If the tests still pass, the mutant **survived**: a real bug of that shape would not be caught.

## Setup

- Stryker is a local tool: `.config/dotnet-tools.json` pins `dotnet-stryker`. Restore with `dotnet tool restore`.
- One `stryker-config.json` per test project that is mutated, starting with the domain tests:

```json
{
  "stryker-config": {
    "project": "<App>.Domain.csproj",
    "test-projects": ["<App>.Domain.Tests.csproj"],
    "reporters": ["progress", "html", "json"],
    "thresholds": { "high": 90, "low": 75, "break": 0 }
  }
}
```

`break: 0` means the score is measured, not enforced, until thresholds are agreed from experiment data.

## When it runs

| Tier | Command | Scope |
| --- | --- | --- |
| Agent loop | not run (too slow) | |
| Pull request | `dotnet stryker --since:main` | Only mutants in changed lines |
| Nightly | `dotnet stryker` | Whole domain |

Run it inside Docker like every other check (see `docker`).

## Reading the result

- Record the **mutation score** and the counts of killed, survived, timeout and no-coverage mutants.
- For each survivor, decide:
  1. **Missing test.** Write a property (preferred) or an example that kills it. Re-run.
  2. **Equivalent mutant.** The change cannot alter behaviour (for example a redundant check). Simplify the code so the mutant disappears, or leave it and say why in the commit message. Never disable a mutator globally to raise the score.
- A survivor inside a domain guard is a bug in the tests. Fix it before opening the pull request.

## Never

- Raise the score by excluding files, lines or mutators without a written reason.
- Delete or weaken a test to make the run faster.
