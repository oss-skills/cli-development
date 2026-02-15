---
name: cli-development
description: >
  Design and build production-grade CLI tools that wrap REST APIs. Covers architecture
  (Kong struct-tag commands, layered internal packages), auth flows (OAuth2, keyring,
  multi-account), output formatting (JSON/plain/rich with stdout/stderr separation),
  agent-friendly design (stable exit codes, schema introspection, command allowlisting),
  and extensibility patterns. Use when building any Go CLI that wraps external APIs,
  designing CLI command hierarchies, implementing OAuth2 flows for CLI tools, or when
  the user mentions "cli-development", "CLI architecture", or "API wrapper CLI".
  Inspired by gogcli by Peter Steinberger (https://github.com/steipete/gogcli).
---

# CLI Development

Production-grade patterns for building Go CLI tools that wrap REST APIs.
Distilled from [gogcli](https://github.com/steipete/gogcli) by [Peter Steinberger](https://github.com/steipete).

## Core Philosophy

1. **Data goes to stdout, everything else to stderr** — the iron rule
2. **Three consumers, three modes** — humans (rich), scripts (plain/TSV), agents (JSON)
3. **Desire paths** — honor how users naturally type, not just how you organize
4. **Least privilege** — request only what you need, support read-only modes
5. **Fail loudly, exit precisely** — typed errors map to stable, documented exit codes
6. **Extensibility by addition** — new services = new packages, zero changes to existing code

## Architecture

### Project Layout

```
cmd/<binary>/main.go       # Minimal: cmd.Execute(os.Args[1:])
internal/
  cmd/                     # Kong command structs + Run() methods
    root.go                # CLI struct, RootFlags, Execute(), parser setup
    exit_codes.go          # Stable exit code constants
    <service>.go           # One file per service (admin.go, mail.go)
  <api>/                   # API client wrappers per service
  auth/                    # OAuth2 flow, token management, scope registry
  config/                  # XDG config, JSON5, account/client mappings
  secrets/                 # Keyring abstraction (OS keychain + file fallback)
  outfmt/                  # Output formatting (JSON/plain/rich modes)
  ui/                      # Terminal colors, printers (via context)
  errfmt/                  # Error formatting, typed errors
```

Each layer depends only on layers below it. Commands never import other commands.

### Kong Command Patterns

Use struct tags for the entire command tree. See `./references/kong-patterns.md` for detailed examples.

**Key patterns:**
- Root CLI struct embeds service commands AND desire-path shortcuts
- Service commands group subcommands by verb category (`group:"read"`, `group:"write"`)
- Desire paths are `hidden:""` — functional but not cluttering default help
- Every leaf command has `Run(ctx context.Context, flags *RootFlags) error`
- Global flags live in `RootFlags` struct embedded at root level

### Desire Paths

Maintain **two routes** to every common action:

| Canonical (discoverable) | Shortcut (fast) |
|--------------------------|-----------------|
| `cli mail send` | `cli send` |
| `cli drive ls` | `cli ls` |
| `cli auth login` | `cli login` |

Shortcuts are hidden from default help. Reveal with env var (e.g., `CLI_HELP=full`).

## Output Formatting

| Mode | Flag | Target | Format |
|------|------|--------|--------|
| Rich | default | Interactive TTY | Colored tables, hints on stderr |
| Plain | `--plain` | Piping | TSV, no colors |
| JSON | `--json` | Scripting/agents | Structured JSON to stdout |

Store mode in `context.Context`, retrieve via `outfmt.FromContext(ctx)`.
Auto-detect TTY for colors. Respect `NO_COLOR` env. Use `muesli/termenv`.

## Agent-Friendly Design

Essential features for LLM agent and automation consumers:

| Feature | Implementation |
|---------|---------------|
| Stable exit codes | Documented, machine-readable (0=success, 4=auth, 5=not found, 7=rate limited) |
| `--results-only` | Strip envelope, return just data array (hidden flag) |
| `--select` | Project JSON to specific fields (hidden flag) |
| `--no-input` | Never prompt, fail instead |
| `--dry-run` | Show what would happen without executing (hidden flag) |
| `--force` | Skip confirmations for destructive ops |
| Schema introspection | `cli schema [cmd]` emits command tree as JSON |
| Command allowlist | `--enable-commands a,b` restricts available commands |

See `./references/agent-design.md` for exit code table and implementation details.

## Auth Flow

**Selection chain:** `--account` flag -> env var -> keyring default -> single stored token -> error

**Keyring backend chain:** env override -> config setting -> auto-detect (macOS Keychain > Linux Secret Service > encrypted file)

**Key principles:**
- Store refresh tokens, not access tokens
- Support multiple accounts with aliases
- Scope management: per-service scopes, `--readonly` flag
- Custom token header support (not all APIs use `Bearer`)
- File-based fallback for headless/WSL/container environments

See `./references/auth-patterns.md` for implementation details.

## Error Handling

Use **typed errors** with checker functions:

```go
type RateLimitError struct { RetryAfter time.Duration; Attempt int }
type AuthRequiredError struct { Service, Account string; Cause error }
type NotFoundError struct { ResourceID string }

func IsRateLimitError(err error) bool { ... }  // uses errors.As
```

Map errors to stable exit codes in one place. Map HTTP status codes:
401->4 (auth), 403->6 (permission), 404->5 (not found), 429->7 (rate limit), 5xx->8 (retryable).

## Config Management

- **Location:** XDG-compliant paths per platform
- **Format:** JSON5 (supports comments, trailing commas)
- **CLI:** `cli config get/set/unset/list/keys/path`
- **Secrets separate from config** — credentials in keyring, mappings in config

## Testing Strategy

| Layer | Approach | Requirement |
|-------|----------|-------------|
| Unit | `testing` + `testify` + `httptest` | Always |
| Integration | Build-tagged, real API calls | Opt-in |
| Live scripts | Shell scripts per service | Manual |

Interfaces at service boundaries enable mocking. Module-level function vars allow test substitution.

## Adding New Services

Adding a service should be mechanical, not inventive:

1. Create API client package (`internal/<api>/<service>/`)
2. Register scopes in auth layer
3. Create Kong command struct (`internal/cmd/<service>.go`)
4. Embed in root CLI struct (one line)
5. Optionally add desire-path shortcuts
6. Add tests

Zero changes to existing code. See `./references/extensibility.md`.

## Build & Release

- **Makefile:** `build`, `fmt`, `lint`, `test`, `ci` (full gate)
- **GoReleaser:** Cross-platform binaries, CGO for macOS Keychain
- **Dev tools:** Pinned in `.tools/` (goimports, gofumpt, golangci-lint)
- **Git hooks:** lefthook for pre-commit/pre-push

## Live API Testing

When wrapping a third-party REST API, **never trust the docs for response shapes**. Probe the real API first.

### The Problem

API documentation frequently lies about:
- **Field types** — docs say `int`, API returns `"123"` (quoted string)
- **Response envelopes** — docs say `{"data": [items]}`, API returns `{"data": {"items": [], "count": 0}}`
- **Boolean encoding** — docs say `true/false`, API returns `"0"/"1"`
- **Error responses** — API returns errors with HTTP 200 and an `"error"` field in the JSON body
- **Auth styles** — API rejects standard OAuth2 Basic Auth, requires params in POST body
- **Scope names** — documented scopes don't exist; actual scopes use different naming conventions
- **Endpoint access** — some endpoints require partner-level access not mentioned in public docs

### Probe-First Workflow

Before defining Go struct types for any API response:

1. **Get a valid token** and curl the endpoint directly
2. **Dump the raw JSON** — check actual field types, nesting, and envelope structure
3. **Define Go types from the real response**, not the docs
4. **Test edge cases** — empty lists, single-item responses, error states
5. **Check if numeric fields come as strings** — common in APIs that evolved from XML/SOAP

```bash
# Probe pattern: dump raw response and inspect field types
TOKEN="..."
curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/endpoint" \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
for k, v in sorted(data.items()):
    print(f'{k} ({type(v).__name__}): {repr(v)[:100]}')
"
```

### Defensive Type Patterns

```go
// BAD: trusting docs that say "integer"
type Message struct {
    ReceivedTime  int64 `json:"receivedTime"`
    HasAttachment bool  `json:"hasAttachment"`
    Priority      int   `json:"priority"`
}

// GOOD: using string for fields that might be quoted
type Message struct {
    ReceivedTime  string `json:"receivedTime"`  // "1771122930710"
    HasAttachment string `json:"hasAttachment"` // "0" or "1"
    Priority      string `json:"priority"`      // "3"
}
// Parse in the CLI display layer, not the API types
```

### Token Exchange Gotchas

Always check for error fields in token responses regardless of HTTP status:

```go
// BAD: only checking HTTP status
if resp.StatusCode != http.StatusOK {
    return nil, fmt.Errorf("HTTP %d", resp.StatusCode)
}
json.NewDecoder(resp.Body).Decode(&tokenResp) // may have empty access_token

// GOOD: parse first, then check for errors
json.NewDecoder(resp.Body).Decode(&tokenResp)
if tokenResp.Error != "" {
    return nil, fmt.Errorf("token error: %s", tokenResp.Error)
}
if tokenResp.AccessToken == "" {
    return nil, fmt.Errorf("empty access token in response")
}
```

### E2E Smoke Test Script

Build a shell script that exercises every read-only endpoint and reports pass/fail/skip:

```bash
#!/bin/bash
pass=0; fail=0; skip=0

run_test() {
    local name="$1"; shift
    output=$("$@" 2>&1)
    rc=$?
    if [ $rc -eq 0 ]; then
        echo "PASS  $name"; ((pass++))
    elif echo "$output" | grep -q "not available\|upgrade\|permission"; then
        echo "SKIP  $name (plan tier)"; ((skip++))
    else
        echo "FAIL  $name (exit $rc)"; ((fail++))
        echo "      $output" | head -1
    fi
}

run_test "users list"    ./cli admin users list
run_test "groups list"   ./cli admin groups list
run_test "messages list" ./cli mail messages list --limit 1
# ... add all endpoints

echo -e "\nResults: $pass passed, $fail failed, $skip skipped"
```

## References

| File | When to Read |
|------|-------------|
| `./references/kong-patterns.md` | Designing command structs and flag patterns |
| `./references/auth-patterns.md` | Implementing OAuth2 flows and keyring storage |
| `./references/agent-design.md` | Making CLI agent/automation-friendly |
| `./references/extensibility.md` | Adding new API services to an existing CLI |

---
*Credits: Patterns distilled from [gogcli](https://github.com/steipete/gogcli) by [Peter Steinberger](https://github.com/steipete). Licensed as open knowledge for the CLI community.*
