Experiment repository for trying out [GitHub CLI's Agent Skills management feature](https://github.blog/changelog/2026-04-16-manage-agent-skills-with-github-cli/).

## Install

Install the [`review-memory`](skills/review-memory/SKILL.md) skill:

```sh
gh skill install gawakawa/gh-skill review-memory --agent claude-code --scope user
```

| Option | Description |
| --- | --- |
| `--agent` | Target agent, e.g. `claude-code`, `codex`. |
| `--scope` | `project` or `user`. |
