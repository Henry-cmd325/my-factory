# Factory skills

Skills tell coding agents how code must be written in a factory project. Each skill is a folder with a `SKILL.md`: rules to follow, and the checks that prove them.

| Skill | Covers | Status |
| --- | --- | --- |
| [`domain-modeling`](domain-modeling/SKILL.md) | Requirements to C# domain classes, every rule as a guard | First version (inventory run 1) |
| [`vertical-slice`](vertical-slice/SKILL.md) | AOT-safe slice anatomy on Clean Architecture + VSA, Minimal APIs | First version (inventory run 1) |
| [`property-testing`](property-testing/SKILL.md) | CsCheck properties for every domain rule, saved seeds | First version (inventory run 1) |
| [`mutation-testing`](mutation-testing/SKILL.md) | Stryker.NET, reading survivors, tiers | First version (inventory run 1) |
| [`docker`](docker/SKILL.md) | App, database and checks in Docker Compose | First version (inventory run 1) |
| `fastsharp` | Modules and endpoints with FastSharp | Waiting for an AOT-clean FastSharp release |
| `ui-testing` | Agent UI exploration turned into Playwright scripts | Not started (needs a frontend) |

## Order for a new feature

1. `domain-modeling`: write or change the domain classes and their guards.
2. `property-testing`: write properties for every guard.
3. `vertical-slice`: build the slice, its store and its HTTP tests.
4. `docker`: run the checks in Compose.
5. `mutation-testing`: measure the score and kill survivors.

## Using the skills in a project

Copy the skill folders into the project's `.claude/skills/` (Claude Code) and note the factory commit they came from. Other agents read the same Markdown files.
