# Qlty Skills

Prompts and skills for AI coding tools to help you set up and manage [Qlty](https://qlty.sh) workflows.

## Available Skills

| Skill | Description |
|-------|-------------|
| [setup-coverage](skills/setup-coverage/SKILL.md) | Set up code coverage reporting in CI using Qlty Cloud |

## Installation

### Claude Code

```bash
# Add the Qlty marketplace
/plugin marketplace add qltysh/qlty-skills

# Install the plugin
/plugin install qlty-skills@qlty
```

Skills are then available as `/qlty-skills:setup-coverage`.

### Cursor

Copy the skill file into your Cursor rules directory:

```bash
cp skills/setup-coverage/SKILL.md .cursor/rules/setup-coverage.md
```

### Windsurf

Add the skill as a Windsurf rule by copying the contents of the relevant `SKILL.md` file into your Windsurf rules configuration.

### Other AI Coding Tools

Use the contents of any `SKILL.md` file as a system prompt or instruction file in your tool of choice. The frontmatter (between the `---` markers) is Claude Code-specific metadata and can be ignored — the instructions below it are universal.

## License

This project is licensed under the MIT License — see [LICENSE](LICENSE) for details.

## Contributing

Each skill lives in its own directory under `skills/` with a `SKILL.md` file. To add a new skill, create a new directory and follow the existing pattern.
