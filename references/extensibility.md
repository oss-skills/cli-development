# Extensibility Patterns

How to design a CLI so adding new API services is mechanical, not inventive.

## Table of Contents

- [The Extensibility Goal](#the-extensibility-goal)
- [Adding a New Service Checklist](#adding-a-new-service-checklist)
- [API Client Package Pattern](#api-client-package-pattern)
- [Scope Registration](#scope-registration)
- [Command Struct Pattern](#command-struct-pattern)
- [Root Integration](#root-integration)
- [Testing Pattern](#testing-pattern)
- [Contributor Guide Template](#contributor-guide-template)

## The Extensibility Goal

Adding a new API service should require:
- **2 new packages** (API client + commands)
- **1 line change** in root CLI struct
- **0 changes** to auth, config, output, or other services

If adding a service requires modifying the auth layer, output layer, or existing services, the architecture has a coupling problem.

## Adding a New Service Checklist

```
1. [ ] Create API client: internal/<api>/<service>/client.go
2. [ ] Register scopes: internal/auth/scopes.go (add const + scope strings)
3. [ ] Create commands: internal/cmd/<service>.go
4. [ ] Embed in root: internal/cmd/root.go (one line)
5. [ ] Add desire paths: internal/cmd/root.go (optional, hidden shortcuts)
6. [ ] Add tests: internal/cmd/<service>_test.go
7. [ ] Add live test: scripts/live-tests/<service>.sh (optional)
```

## API Client Package Pattern

Each service gets its own package under the API client directory:

```go
// internal/zohoapi/admin/client.go
package admin

import "context"

type Client struct {
    baseURL    string
    httpClient *http.Client
    orgID      string
}

func New(httpClient *http.Client, baseURL, orgID string) *Client {
    return &Client{
        baseURL:    baseURL,
        httpClient: httpClient,
        orgID:      orgID,
    }
}

// Each API operation is a method on Client
func (c *Client) ListUsers(ctx context.Context, opts ListUsersOptions) (*UserList, error) {
    // Build request, execute, parse response
}

func (c *Client) GetUser(ctx context.Context, userID string) (*User, error) { ... }
func (c *Client) CreateUser(ctx context.Context, user *CreateUserRequest) (*User, error) { ... }
```

**Key rules:**
- Package per service, not one giant client
- Accept `*http.Client` (with auth transport already wired) — don't handle auth internally
- Accept base URL — don't hardcode regions
- Return typed responses, not raw JSON
- Error types from shared errors package

## Scope Registration

Centralized scope registry:

```go
// internal/auth/scopes.go
const (
    ServiceAdmin Service = "admin"
    ServiceMail  Service = "mail"
    ServiceCRM   Service = "crm"  // New service just adds a const + entry
)

var registry = map[Service]ServiceInfo{
    ServiceAdmin: {
        Scopes:   []string{"ZohoAdmin.users.READ", "ZohoAdmin.groups.READ"},
        ReadOnly: []string{"ZohoAdmin.users.READ", "ZohoAdmin.groups.READ"},
    },
    ServiceMail: {
        Scopes:   []string{"ZohoMail.messages.ALL", "ZohoMail.accounts.READ"},
        ReadOnly: []string{"ZohoMail.messages.READ", "ZohoMail.accounts.READ"},
    },
    // Adding CRM = just add an entry here
    ServiceCRM: {
        Scopes:   []string{"ZohoCRM.modules.ALL"},
        ReadOnly: []string{"ZohoCRM.modules.READ"},
    },
}
```

## Command Struct Pattern

Follow the same structure for every service:

```go
// internal/cmd/crm.go  (new service)
package cmd

type CRMCmd struct {
    Contacts CRMContactsCmd `cmd:"" name:"contacts" help:"Manage contacts" group:"data"`
    Deals    CRMDealsCmd    `cmd:"" name:"deals" help:"Manage deals" group:"data"`
    Settings CRMSettingsCmd `cmd:"" name:"settings" help:"CRM settings" group:"admin"`
}

type CRMContactsCmd struct {
    List   CRMContactsListCmd   `cmd:"" name:"list" aliases:"ls" help:"List contacts"`
    Get    CRMContactsGetCmd    `cmd:"" name:"get" help:"Get contact by ID"`
    Create CRMContactsCreateCmd `cmd:"" name:"create" help:"Create contact"`
}

type CRMContactsListCmd struct {
    Max    int    `name:"max" default:"20" help:"Max results"`
    Page   string `name:"page" help:"Page token"`
    Fields string `name:"fields" help:"Fields to return"`
}

func (c *CRMContactsListCmd) Run(ctx context.Context, flags *RootFlags) error {
    client, err := crmClientFromContext(ctx)
    if err != nil {
        return err
    }
    contacts, err := client.ListContacts(ctx, crm.ListOptions{Max: c.Max})
    if err != nil {
        return err
    }
    return outfmt.Write(ctx, contacts)
}
```

## Root Integration

One line to add a new service:

```go
// internal/cmd/root.go
type CLI struct {
    RootFlags

    // Services
    Auth   AuthCmd   `cmd:"" name:"auth" group:"services"`
    Admin  AdminCmd  `cmd:"" name:"admin" group:"services"`
    Mail   MailCmd   `cmd:"" name:"mail" group:"services"`
    CRM    CRMCmd    `cmd:"" name:"crm" group:"services"`  // <-- ONE LINE

    // Desire paths (optional for new service)
    Contacts CRMContactsListCmd `cmd:"" name:"contacts" group:"shortcuts" hidden:""`
}
```

## Testing Pattern

Mirror the service structure in tests:

```go
// internal/cmd/crm_test.go
func TestCRMContactsList(t *testing.T) {
    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        assert.Equal(t, "/api/v1/contacts", r.URL.Path)
        json.NewEncoder(w).Encode(map[string]any{
            "contacts": []map[string]any{
                {"id": "1", "name": "Test User"},
            },
        })
    }))
    defer srv.Close()

    // Run command against test server
    out := executeCommand(t, "--json", "crm", "contacts", "list",
        "--base-url", srv.URL)

    var result struct {
        Contacts []map[string]any `json:"contacts"`
    }
    require.NoError(t, json.Unmarshal(out, &result))
    assert.Len(t, result.Contacts, 1)
}
```

## Contributor Guide Template

Include in your project for community contributors:

```markdown
## Adding a New Service

1. Create `internal/<api>/<service>/` with `client.go` and types
2. Add scope entry in `internal/auth/scopes.go`
3. Create `internal/cmd/<service>.go` with command structs
4. Add one line to `CLI` struct in `internal/cmd/root.go`
5. Add tests in `internal/cmd/<service>_test.go`
6. Run `make ci` to verify everything passes

### Conventions
- Each command struct has `Run(ctx, *RootFlags) error`
- Use `outfmt.Write(ctx, data)` for output (handles JSON/plain/rich)
- Use typed errors from `internal/errfmt`
- Group subcommands: "read", "write", "admin"
- Add `aliases:"ls"` for common abbreviations
```

---
*Patterns from gogcli by Peter Steinberger*
