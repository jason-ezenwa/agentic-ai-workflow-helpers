# Agents

Each agent is defined in two files — one per tool that loads it:

| File | Used by | Format |
|------|---------|--------|
| `<agent>.md` | Claude | Markdown with YAML frontmatter |
| `<agent>.toml` | Codex | TOML |

## Keeping them in sync

The `developer_instructions` field in each `.toml` file is the equivalent of the markdown body in the `.md` file. When you update an agent's instructions, update both files.

The `.md` frontmatter fields (`name`, `description`) map directly to the TOML `name` and `description` fields — keep these identical.

**`model:`** in the `.md` frontmatter uses Claude model names, while the TOML `model` field uses Codex model names. `model: inherit` in markdown means omit the `model` field in TOML so Codex uses the session default.
