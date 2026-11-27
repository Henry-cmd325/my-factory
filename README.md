# my-factory

An AI-assisted software factory for .NET: start new software fast and keep the quality high.

> Status: early design. The factory changes with every experiment. The first experiment lives in [inventory-lab](https://github.com/Henry-cmd325/inventory-lab).

## The idea

AI agents write the code. The factory is the set of **skills** that tells them *how* the code must be written, and the **quality gates** that decide whether the result is good enough.

1. You describe what the software needs, in plain language (a prompt or a GitHub issue).
2. An agent writes or updates the domain as plain C# classes. You review this model.
3. The agent follows the factory skills to build vertical slices: endpoint, handler, validation, mapping, persistence.
4. The agent writes property-based tests and Playwright UI scripts for each slice.
5. Quality gates run. A failed gate goes back to the agent with the report, until everything passes.

## Defaults

| Area | Default |
| --- | --- |
| Backend | .NET, AOT-first (ready for when EF Core fully supports Native AOT) |
| Architecture | Clean Architecture + Vertical Slice Architecture (VSA) |
| Endpoints and structure | [FastSharp](https://github.com/Henry-cmd325/FastSharp) |
| Frontend | Any language, also organised by vertical slices |
| Runtime | Everything runs on Docker: any agent can start the project quickly, and it can be deployed anywhere |
| Agents | Agent-agnostic; Claude Code first |

## Principles

- **The models are the documentation.** There is no separate spec layer (Markdown or executable) that can drift from the code.
- **History, not specs.** Decisions and their reasons live in agent memory ([Engram](https://github.com/Gentleman-Programming/engram) Cloud, shared by all agents) and in [`CHANGELOG.md`](CHANGELOG.md), generated from [Conventional Commits](https://www.conventionalcommits.org/).
- **Property-based testing** and **mutation testing** are core: tests must prove they catch bugs.
- **Agents explore, scripts guard.** Agents explore the UI; every finding becomes a deterministic Playwright script.
- **Deterministic simulation testing** comes later, for concurrency and I/O.

## When tests run

Slow checks run less often. A change inside one slice runs only that slice's tests; a change in shared code runs everything.

| Tier | When | What | Budget |
| --- | --- | --- | --- |
| T0: Agent loop | Every agent edit | AOT build, example and property tests of the changed slice, saved failing seeds, Playwright scripts for the changed slice's screens | Seconds; about 1 minute with UI scripts |
| T1: Pull request | Every push to a PR | All tests of affected slices, full property runs, mutation testing on changed lines, API snapshots, scripted UI tests of affected screens | Under 10 minutes |
| T2: Main / nightly | Merge to main or nightly | Everything, new random seeds, full mutation run, simulation, all critical UI flows, one exploratory UI run | Under 1 hour |
| T3: Weekly / release | Weekly and before a release | Long simulation campaigns, full exploratory UI runs, mutation score trend | Hours |

Every failure found by a slow tier is saved and replayed at T0 from then on.

## Repository layout

| Path | What lives there |
| --- | --- |
| [`skills/`](skills/) | The factory skills that agents follow |
| [`AGENTS.md`](AGENTS.md) | Rules for any agent working on this repo |
| [`CHANGELOG.md`](CHANGELOG.md) | Human-readable history |

## License

[Apache License 2.0](LICENSE)
