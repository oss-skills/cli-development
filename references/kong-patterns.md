# Kong Command Patterns

Detailed patterns for structuring Go CLI commands with the Kong framework.

## Table of Contents

- [Root CLI Struct](#root-cli-struct)
- [Service Command Struct](#service-command-struct)
- [Leaf Command Struct](#leaf-command-struct)
- [RootFlags (Global Flags)](#rootflags)
- [Desire Path Wiring](#desire-path-wiring)
- [Kong Struct Tag Reference](#kong-struct-tag-reference)
- [Dependency Injection via BeforeApply](#dependency-injection)
- [Execute Function](#execute-function)

## Root CLI Struct

The root struct defines the entire command tree declaratively:

```go
type CLI struct {
    RootFlags

    Version kong.VersionFlag `name:"version" help:"Print version"`

    // Desire path shortcuts (hidden from default help)
    Send   MailSendCmd   `cmd:"" name:"send" help:"Send message" group:"shortcuts" hidden:""`
    Ls     AdminLsCmd    `cmd:"" name:"ls" help:"List resources" group:"shortcuts" hidden:""`
    Login  AuthLoginCmd  `cmd:"" name:"login" help:"Authenticate" group:"shortcuts" hidden:""`
    Logout AuthLogoutCmd `cmd:"" name:"logout" help:"Remove credentials" group:"shortcuts" hidden:""`

    // Service hierarchy (canonical commands)
    Auth   AuthCmd   `cmd:"" name:"auth" help:"Authentication management" group:"services"`
    Admin  AdminCmd  `cmd:"" name:"admin" help:"Organization admin" group:"services"`
    Mail   MailCmd   `cmd:"" name:"mail" help:"Email operations" group:"services"`
    Config ConfigCmd `cmd:"" name:"config" help:"Configuration" group:"util"`

    // Agent utilities (hidden)
    Schema SchemaCmd `cmd:"" name:"schema" group:"util" hidden:""`
    Agent  AgentCmd  `cmd:"" name:"agent" group:"util"`
}
```

## Service Command Struct

Group subcommands by verb category:

```go
type MailCmd struct {
    // Read operations
    Search   MailSearchCmd   `cmd:"" name:"search" help:"Search messages" group:"read"`
    Get      MailGetCmd      `cmd:"" name:"get" help:"Get message by ID" group:"read"`
    List     MailListCmd     `cmd:"" name:"list" aliases:"ls" help:"List messages" group:"read"`
    Folders  MailFoldersCmd  `cmd:"" name:"folders" help:"List folders" group:"read"`

    // Write operations
    Send     MailSendCmd     `cmd:"" name:"send" help:"Send message" group:"write"`
    Move     MailMoveCmd     `cmd:"" name:"move" help:"Move messages" group:"write"`
    Delete   MailDeleteCmd   `cmd:"" name:"delete" help:"Delete messages" group:"write"`

    // Settings
    Settings MailSettingsCmd `cmd:"" name:"settings" help:"Mail settings" group:"admin"`
    Filters  MailFiltersCmd  `cmd:"" name:"filters" help:"Mail filters" group:"admin"`
}
```

## Leaf Command Struct

Each leaf command defines its flags/args and implements `Run`:

```go
type MailSearchCmd struct {
    Query   string `arg:"" help:"Search query"`
    Max     int    `name:"max" aliases:"limit" default:"20" help:"Maximum results"`
    Page    string `name:"page" aliases:"cursor" help:"Pagination cursor"`
    Folder  string `name:"folder" default:"inbox" help:"Folder to search"`
    After   string `name:"after" help:"Messages after date"`
    Before  string `name:"before" help:"Messages before date"`
}

func (c *MailSearchCmd) Run(ctx context.Context, flags *RootFlags) error {
    client, err := apiClientFromContext(ctx)
    if err != nil {
        return err
    }

    results, err := client.Mail().Search(ctx, c.Query, mail.SearchOptions{
        Max:    c.Max,
        Folder: c.Folder,
        After:  c.After,
        Before: c.Before,
    })
    if err != nil {
        return err
    }

    return outfmt.Write(ctx, results)
}
```

## RootFlags

Global flags available to all commands:

```go
type RootFlags struct {
    Color          string `name:"color" default:"auto" enum:"auto,always,never" help:"Color output"`
    Account        string `name:"account" env:"CLI_ACCOUNT" help:"Account to use"`
    JSON           bool   `name:"json" env:"CLI_JSON" help:"JSON output"`
    Plain          bool   `name:"plain" env:"CLI_PLAIN" help:"Plain text output"`
    ResultsOnly    bool   `name:"results-only" hidden:"" help:"Strip JSON envelope"`
    Select         string `name:"select" hidden:"" help:"Project JSON fields"`
    DryRun         bool   `name:"dry-run" hidden:"" help:"Show what would happen"`
    Force          bool   `name:"force" help:"Skip confirmations"`
    NoInput        bool   `name:"no-input" help:"Never prompt for input"`
    Verbose        bool   `name:"verbose" help:"Verbose output"`
}
```

## Desire Path Wiring

Shortcuts point to the same command structs as canonical paths:

```go
// In root CLI struct, both reference the same type:
Send MailSendCmd `cmd:"" name:"send" group:"shortcuts" hidden:""`
// ...
// Inside MailCmd:
Send MailSendCmd `cmd:"" name:"send" group:"write"`
```

The `hidden:""` tag keeps shortcuts out of default `--help` but fully functional.
Reveal all commands with an env var: `CLI_HELP=full cli --help`.

## Kong Struct Tag Reference

| Tag | Purpose | Example |
|-----|---------|---------|
| `cmd:""` | Marks field as subcommand | `Ls LsCmd \`cmd:""\`` |
| `arg:""` | Positional argument | `Query string \`arg:""\`` |
| `name:"..."` | Command/flag name | `name:"max"` |
| `aliases:"..."` | Alternative names | `aliases:"ls,list"` |
| `help:"..."` | Help text | `help:"Max results"` |
| `default:"..."` | Default value | `default:"20"` |
| `enum:"..."` | Allowed values | `enum:"auto,always,never"` |
| `env:"..."` | Env var override | `env:"CLI_ACCOUNT"` |
| `group:"..."` | Help group | `group:"read"` |
| `hidden:""` | Hide from help | `hidden:""` |
| `embed:""` | Embed struct flags | `embed:""` |

## Dependency Injection

Use Kong's `BeforeApply` hook for wiring dependencies:

```go
func (f *RootFlags) BeforeApply(ctx *kong.Context) error {
    // Load config
    cfg, err := config.Load()
    if err != nil {
        return err
    }

    // Set up output mode
    mode := outfmt.Mode{JSON: f.JSON, Plain: f.Plain}

    // Wire into context
    goCtx := context.Background()
    goCtx = config.WithContext(goCtx, cfg)
    goCtx = outfmt.WithContext(goCtx, mode)

    ctx.Bind(goCtx)
    return nil
}
```

## Execute Function

The main entrypoint wires Kong parser and runs:

```go
func Execute(args []string) int {
    var cli CLI

    parser, err := kong.New(&cli,
        kong.Name("zoh"),
        kong.Description("Zoho in your terminal"),
        kong.Vars{"version": version},
        kong.UsageOnError(),
    )
    if err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        return 1
    }

    kctx, err := parser.Parse(args)
    if err != nil {
        parser.FatalIfErrorf(err)
        return 2
    }

    if err := kctx.Run(); err != nil {
        return stableExitCode(err)
    }
    return 0
}
```

---
*Patterns from gogcli by Peter Steinberger*
