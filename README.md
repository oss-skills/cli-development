# cli-development

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) for building Go CLI tools that wrap REST APIs.

## Why

Most enterprise software has a REST API but no CLI. That means no scripting, no piping, no automation, no agents. This skill teaches Claude Code how to build the CLI so you don't have to explain the architecture every time.

Covers project layout, Kong commands, OAuth2 with keyring, output formatting (rich/plain/JSON), agent-friendly flags, and how to add new API services without touching existing code.

Based on patterns from [gogcli](https://github.com/steipete/gogcli) by [Peter Steinberger](https://github.com/steipete).

## Install

```bash
npx skills add oss-skills/cli-development
```

Or copy manually:
```bash
cp -r . ~/.claude/skills/cli-development/
```

## What's in here

The skill activates when you ask Claude Code to build a CLI, design commands, or implement OAuth2 for a CLI tool.

```
SKILL.md                          # Main instructions
references/
  kong-patterns.md                # Command struct patterns
  auth-patterns.md                # OAuth2 + keyring
  agent-design.md                 # Stable exit codes, --json, schema introspection
  extensibility.md                # Adding new API services
```

## License

MIT
