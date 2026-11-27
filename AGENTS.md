# Agent rules for my-factory

This repository holds the factory itself: the skills that tell coding agents how to build .NET software, and the quality gates that check the result. It does not hold application code; experiments live in their own repositories (for example [inventory-lab](https://github.com/Henry-cmd325/inventory-lab)).

## Working here

- Write in English.
- Use [Conventional Commits](https://www.conventionalcommits.org/) (`feat:`, `fix:`, `docs:`, `chore:` ...). `CHANGELOG.md` is built from them.
- Keep the skills agent-agnostic: plain Markdown that any agent can read. Claude Code is the first target.
- Do not add a separate specification layer. The C# models are the documentation; decisions go to agent memory and the changelog.

## Skills

Each skill is a folder under `skills/` with a `SKILL.md` file. A skill states rules the agent must follow and the checks that prove them. Change a skill only when an experiment shows why, and say what the experiment showed in the commit message.
