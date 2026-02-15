# Auth Patterns

OAuth2 authentication and credential management patterns for CLI tools.

## Table of Contents

- [Auth Architecture](#auth-architecture)
- [Account Selection Chain](#account-selection-chain)
- [OAuth2 Flow for CLIs](#oauth2-flow-for-clis)
- [Token Management](#token-management)
- [Keyring Abstraction](#keyring-abstraction)
- [Scope Management](#scope-management)
- [Multi-Account Support](#multi-account-support)

## Auth Architecture

```
┌──────────────────────────────────────────────────┐
│                   Command Layer                    │
│   account := resolveAccount(flags, config)         │
│   client := apiClient(ctx, account)                │
└────────────────────┬─────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────┐
│                   Auth Layer                       │
│   tokenSource := tokenSourceForAccount(account)    │
│   httpClient := oauth2.NewClient(ctx, tokenSource) │
└────────────────────┬─────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────┐
│                  Secrets Layer                      │
│   store.GetToken(account) → refreshToken           │
│   store.SetToken(account, token)                   │
└──────────────────────────────────────────────────┘
```

## Account Selection Chain

Resolve account in priority order:

```go
func resolveAccount(flags *RootFlags, cfg *config.File) (string, error) {
    // 1. Explicit flag
    if flags.Account != "" {
        return resolveAlias(flags.Account, cfg)
    }
    // 2. Environment variable
    if env := os.Getenv("CLI_ACCOUNT"); env != "" {
        return resolveAlias(env, cfg)
    }
    // 3. Config default
    if cfg.DefaultAccount != "" {
        return cfg.DefaultAccount, nil
    }
    // 4. Single stored account (auto-select)
    accounts, _ := store.ListAccounts()
    if len(accounts) == 1 {
        return accounts[0], nil
    }
    // 5. Error
    return "", &AuthRequiredError{Message: "no account configured"}
}
```

## OAuth2 Flow for CLIs

Three flow variants for different environments:

### Interactive (default)
1. Open browser to auth URL
2. Start localhost HTTP server for callback
3. Receive auth code via redirect
4. Exchange code for tokens
5. Store refresh token in keyring

### Manual (`--manual`)
1. Print auth URL to stderr
2. User opens URL in any browser
3. User pastes redirect URL back to CLI
4. Exchange code for tokens

### Remote/Headless (`--remote`)
Two-step flow for headless servers:
```bash
# Step 1: On headless server — prints auth URL
cli auth login --remote --step 1

# Step 2: On machine with browser — completes auth
cli auth login --remote --step 2 --code <auth_code>
```

## Token Management

```go
type TokenStore interface {
    GetToken(account string) (*oauth2.Token, error)
    SetToken(account string, token *oauth2.Token) error
    DeleteToken(account string) error
    ListAccounts() ([]string, error)
}

// Custom TokenSource that persists refreshed tokens
type persistingTokenSource struct {
    base    oauth2.TokenSource
    store   TokenStore
    account string
    mu      sync.Mutex
}

func (s *persistingTokenSource) Token() (*oauth2.Token, error) {
    s.mu.Lock()
    defer s.mu.Unlock()

    token, err := s.base.Token()
    if err != nil {
        return nil, err
    }

    // Persist if refresh produced new token
    if err := s.store.SetToken(s.account, token); err != nil {
        // Log but don't fail — token is still valid
        log.Printf("warning: failed to persist token: %v", err)
    }
    return token, nil
}
```

**Key rules:**
- Store refresh tokens, never just access tokens
- Auto-refresh transparently via custom `TokenSource`
- Persist new tokens after refresh (refresh tokens can rotate)
- Support custom auth headers (not all APIs use `Bearer` prefix)
- Token cache with file locking for concurrent CLI invocations

## Keyring Abstraction

```go
type SecretStore struct {
    backend keyring.Keyring
}

func NewSecretStore(cfg *config.File) (*SecretStore, error) {
    backend := cfg.KeyringBackend // "auto", "keychain", "file", etc.

    ring, err := keyring.Open(keyring.Config{
        ServiceName:             "cli-name",
        KeychainTrustApplication: true,
        // File-based fallback config
        FileDir:          filepath.Join(configDir, "keys"),
        FilePasswordFunc: keyring.FixedStringPrompt(os.Getenv("CLI_KEYRING_PASSWORD")),
    })
    if err != nil {
        return nil, fmt.Errorf("keyring: %w", err)
    }
    return &SecretStore{backend: ring}, nil
}
```

**Backend chain:** env override -> config -> auto-detect

**Platform support:**
| Platform | Primary | Fallback |
|----------|---------|----------|
| macOS | Keychain (CGO required) | Encrypted file |
| Linux | Secret Service (D-Bus) | Encrypted file |
| Windows | Credential Manager | Encrypted file |
| WSL/Container | — | Encrypted file (requires `CLI_KEYRING_PASSWORD`) |

Always implement the file-based fallback. Headless environments are common for CLI users.

## Scope Management

Register scopes per service:

```go
type Service string

const (
    ServiceMail  Service = "mail"
    ServiceAdmin Service = "admin"
)

var serviceScopes = map[Service][]string{
    ServiceMail:  {"mail.read", "mail.send", "mail.settings"},
    ServiceAdmin: {"admin.users", "admin.groups", "admin.domains"},
}

func ScopesForServices(services []Service, readonly bool) []string {
    var scopes []string
    for _, svc := range services {
        s := serviceScopes[svc]
        if readonly {
            s = readOnlyVariants(s)
        }
        scopes = append(scopes, s...)
    }
    return dedupe(scopes)
}
```

Support `--readonly` flag for least-privilege auth.

## Multi-Account Support

```go
// Config supports account aliases
type File struct {
    AccountAliases map[string]string `json:"account_aliases"` // "work" -> "admin@company.com"
    DefaultAccount string            `json:"default_account"`
}

// CLI commands for account management
// cli auth login user@example.com
// cli auth alias set work user@example.com
// cli auth list [--check]  (--check validates tokens)
// cli auth logout user@example.com
```

---
*Patterns from gogcli by Peter Steinberger*
