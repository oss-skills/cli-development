# Agent-Friendly CLI Design

Patterns for making CLI tools consumable by LLM agents and automation scripts.

## Table of Contents

- [Design Principles](#design-principles)
- [Stable Exit Codes](#stable-exit-codes)
- [JSON Output Filtering](#json-output-filtering)
- [Command Schema Introspection](#command-schema-introspection)
- [Command Sandboxing](#command-sandboxing)
- [Non-Interactive Mode](#non-interactive-mode)
- [Dry-Run Support](#dry-run-support)
- [Convenience Fields](#convenience-fields)

## Design Principles

Agents need three things from a CLI:
1. **Predictable structure** — same output shape every time
2. **Machine-parseable status** — exit codes, not error messages
3. **Discoverable surface** — what commands exist, what flags they take

## Stable Exit Codes

Every error maps to a documented, stable exit code:

```go
const (
    ExitSuccess       = 0   // Operation completed successfully
    ExitError         = 1   // General/unknown error
    ExitUsage         = 2   // Invalid arguments or flags
    ExitEmpty         = 3   // No results found (with --fail-empty)
    ExitAuth          = 4   // Authentication required or failed
    ExitNotFound      = 5   // Requested resource not found
    ExitPermission    = 6   // Permission denied
    ExitRateLimit     = 7   // Rate limited or quota exceeded
    ExitRetryable     = 8   // Transient failure (retry may succeed)
    ExitConfig        = 10  // Configuration error
    ExitCancelled     = 130 // User interrupt (SIGINT)
)
```

**Map HTTP status codes to exit codes:**

```go
func httpExitCode(statusCode int, err error) int {
    switch statusCode {
    case 401:
        return ExitAuth
    case 403:
        if isRateLimitOrQuota(err) {
            return ExitRateLimit
        }
        return ExitPermission
    case 404:
        return ExitNotFound
    case 429:
        return ExitRateLimit
    default:
        if statusCode >= 500 {
            return ExitRetryable
        }
        return ExitError
    }
}
```

**Expose via CLI:**

```bash
$ cli agent exit-codes
Code  Name          Description
0     success       Operation completed successfully
1     error         General error
2     usage         Invalid arguments or flags
4     auth          Authentication required
5     not_found     Resource not found
6     permission    Permission denied
7     rate_limit    Rate limited or quota exceeded
8     retryable     Transient failure, retry may succeed
10    config        Configuration error
130   cancelled     User interrupt
```

## JSON Output Filtering

### --results-only

Strip envelope metadata, return just the data array:

```bash
# Default JSON output
$ cli --json mail list
{"messages": [...], "nextPage": "abc", "total": 42}

# With --results-only
$ cli --json --results-only mail list
[{...}, {...}, {...}]
```

### --select

Project JSON objects to specific fields:

```bash
$ cli --json --select "id,subject,from.email" mail list
[
  {"id": "123", "subject": "Hello", "from": {"email": "a@b.com"}},
  ...
]
```

Supports dot-notation for nested field access.

### Implementation

```go
type JSONTransform struct {
    ResultsOnly bool
    Select      string // comma-separated field paths
}

func WriteJSON(w io.Writer, data any, transform JSONTransform) error {
    if transform.ResultsOnly {
        data = extractResults(data) // unwrap envelope
    }
    if transform.Select != "" {
        data = projectFields(data, strings.Split(transform.Select, ","))
    }
    enc := json.NewEncoder(w)
    enc.SetIndent("", "  ")
    return enc.Encode(data)
}
```

## Command Schema Introspection

Emit the command tree as machine-readable JSON:

```bash
$ cli schema mail send
{
  "name": "send",
  "path": "mail send",
  "help": "Send an email message",
  "flags": [
    {"name": "to", "type": "string", "required": true, "help": "Recipient"},
    {"name": "subject", "type": "string", "required": true, "help": "Subject line"},
    {"name": "body", "type": "string", "help": "Message body"},
    {"name": "attach", "type": "[]string", "help": "File attachments"}
  ],
  "examples": [
    "cli mail send --to user@example.com --subject 'Hello' --body 'Hi there'"
  ]
}
```

This allows agents to discover available operations without parsing `--help` text.

## Command Sandboxing

Restrict which commands an agent can access:

```bash
# Only allow read operations
$ CLI_ENABLE_COMMANDS=mail,admin cli --json mail list

# Agent can use mail and admin, but not auth or config
$ cli --enable-commands mail,admin --json mail list
```

Implementation: Check allowlist before command dispatch. Return `ExitUsage` for disallowed commands.

## Non-Interactive Mode

```bash
# Never prompt — fail instead of asking
$ cli --no-input mail send --to user@example.com
```

When `--no-input` is set:
- Confirmations default to "no" (operation aborted)
- Missing required input returns `ExitUsage`
- Password prompts fail with helpful error

Combine with `--force` to skip confirmations AND proceed:

```bash
# Skip confirmation and execute
$ cli --force --no-input admin users delete user@example.com
```

## Dry-Run Support

```bash
$ cli --dry-run mail send --to user@example.com --subject "Test"
{
  "action": "mail.send",
  "would_execute": {
    "method": "POST",
    "url": "https://mail.zoho.eu/api/accounts/.../messages",
    "body": {"toAddress": "user@example.com", "subject": "Test"}
  }
}
```

Outputs in the current format mode (JSON/plain/rich). Hidden flag — primarily for agent debugging.

## Convenience Fields

Add derived fields that agents find useful without having to compute them:

```go
type Message struct {
    // API fields
    ReceivedTime int64  `json:"receivedTime"`

    // Convenience fields (computed, not from API)
    ReceivedISO  string `json:"receivedISO"`     // ISO 8601 timestamp
    ReceivedAgo  string `json:"receivedAgo"`      // "2 hours ago"
    HasAttach    bool   `json:"hasAttachments"`   // derived from attachments array
}
```

Agents benefit from pre-computed fields that would otherwise require post-processing.

---
*Patterns from gogcli by Peter Steinberger*
