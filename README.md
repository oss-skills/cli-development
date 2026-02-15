# cli-development

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) for building production-grade Go CLI tools that wrap REST APIs.

Distilled from real-world patterns building API wrapper CLIs. Covers architecture, auth flows, output formatting, agent-friendly design, and extensibility.

## Why

Old enterprise software with REST APIs deserves good CLIs. Good CLIs make agentic development possible. This skill teaches your AI coding assistant the patterns that make CLIs production-ready — not tutorial code, but battle-tested architecture.

## What's Inside

- **Architecture** — Kong struct-tag commands, layered `internal/` packages, desire paths
- **Auth** — OAuth2 flows, keyring storage (with file fallback for WSL/containers), multi-account support
- **Output** — Three modes (rich/plain/JSON), stdout/stderr separation, `NO_COLOR` support
- **Agent-friendly design** — Stable exit codes, `--json`, `--results-only`, schema introspection, command allowlisting
- **Live API testing** — Probe-first workflow for APIs whose docs lie about response shapes
- **Extensibility** — Adding new services is mechanical, not inventive

## Install

```bash
# Add to your project
npx skills add oss-skills/cli-development

# Or copy manually
cp -r . ~/.claude/skills/cli-development/
```

## Usage

Once installed, Claude Code automatically activates this skill when you:
- Ask to build a CLI tool
- Design command hierarchies
- Implement OAuth2 flows for CLI tools
- Mention "CLI architecture" or "API wrapper CLI"

## Structure

```
SKILL.md                          # Main skill instructions
references/
  kong-patterns.md                # Kong command struct patterns
  auth-patterns.md                # OAuth2 + keyring implementation
  agent-design.md                 # Agent/automation-friendly features
  extensibility.md                # Adding new API services
```

## Credits

Patterns distilled from [gogcli](https://github.com/steipete/gogcli) by [Peter Steinberger](https://github.com/steipete).

## License

MIT
