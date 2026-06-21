# Contributing

## Adding a skill

Place each skill at `skills/<name>/SKILL.md`. Required frontmatter:

- `name` — must match the directory name and follow the [agentskills.io](https://agentskills.io/specification) naming rules.
- `description` — one line stating what the skill does and when to use it.
- `allowed-tools` — optional; must be a **string** (e.g. `"Bash, Read, Write"`), never an array.

## Publishing

```sh
# Validate against the Agent Skills spec (no publish)
gh skill publish --dry-run

# Publish: adds the `agent-skills` topic and creates a GitHub release
gh skill publish --tag v0.1.0
```

Release tags must be strict semver `vMAJOR.MINOR.PATCH` (e.g. `v0.1.0`).
