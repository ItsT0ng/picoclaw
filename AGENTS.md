# AGENTS.md — Developer Guide for Agentic Coding Agents

This file provides guidance for AI coding agents working in the **PicoClaw** repository.
PicoClaw is an ultra-lightweight personal AI agent written in Go.

---

## Build & Run Commands

```bash
# Build for current platform
make build

# Build for all supported platforms (linux, darwin, windows, arm, riscv64, etc.)
make build-all

# Run (builds then executes)
make run

# Install binary to system
make install
```

Build flags used: `CGO_ENABLED=0 -tags stdjson -ldflags "-s -w ..."`.

---

## Test Commands

```bash
# Run all tests
make test
# Equivalent: go test ./...

# Run tests for a single package
go test ./pkg/providers/...

# Run a single test function
go test ./pkg/providers/... -run TestCreateProvider_ExplicitClaude

# Run with verbose output
go test ./pkg/providers/... -run TestCreateProvider -v

# Run tests with race detector
go test -race ./...

# Before testing, regenerate embedded assets (CI does this too)
go generate ./...
```

---

## Lint & Format Commands

```bash
# Run linter (golangci-lint v2)
make lint

# Auto-fix lint issues
make fix

# Format code
make fmt

# Generate embedded assets / code
make generate
```

The linter is configured in `.golangci.yaml`. Active formatters: `gci`, `gofmt`, `gofumpt`,
`goimports`, `golines`. Max line length is **120 characters**.

---

## Code Style Guidelines

### Imports

Imports are grouped in exactly three sections (enforced by `gci`), separated by blank lines:

```go
import (
    "context"
    "fmt"

    "github.com/spf13/cobra"
    "github.com/stretchr/testify/assert"

    "github.com/sipeed/picoclaw/pkg/config"
    "github.com/sipeed/picoclaw/pkg/logger"
)
```

Order: **standard library → third-party → local module** (`github.com/sipeed/picoclaw/...`).
Never mix groups. `goimports` enforces this automatically.

### Formatting

- Line length: **120 characters** maximum (enforced by `golines`)
- Use `gofumpt` (stricter than `gofmt`): no blank lines at the start/end of blocks
- `interface{}` is rewritten to `any` by gofmt rules — always use `any`
- Use `strings.Builder` for string concatenation in hot paths, not `+` in loops

### Types & Interfaces

- Exported types: `PascalCase` (`AgentLoop`, `ToolRegistry`, `RouteResolver`)
- Interfaces: `PascalCase`, prefer `-er` suffix (`LLMProvider`, `AsyncTool`, `StatefulProvider`)
- Constructors: `NewXxx(...)` pattern returning the type (or pointer) and `error`
- JSON struct tags: `snake_case` (e.g., `json:"model_name"`, `json:"api_key"`)
- Struct fields: exported `PascalCase`, unexported `camelCase`
- Method receivers: short abbreviations, 1–2 letters (e.g., `al` for `AgentLoop`, `r` for `Router`)

### Naming Conventions

- Functions and methods: `camelCase` for unexported, `PascalCase` for exported
- Constants: `PascalCase` for exported (e.g., `FailoverAuth`), typed string constants use plain strings
- Test files: `foo_test.go` in the **same package** (not a `_test`-suffix package)
- Test functions: `TestFoo` or `TestFoo_Scenario` for sub-scenarios
- Mock types: defined in `mock_*_test.go` files within the same package
- Integration tests: `foo_integration_test.go`

### Error Handling

- Always wrap errors: `fmt.Errorf("context description: %w", err)`
- Return errors up the call stack; do not panic except for truly unrecoverable programmer errors
- Tool errors use the result constructor: `tools.ErrorResult("message").WithError(err)`
- Custom error types implement `error` and `Unwrap()` (e.g., `FailoverError`)
- Never silently discard errors; if intentionally ignored, add a comment

### Tool Implementation Pattern

New tools must implement the `Tool` interface in `pkg/tools/`:

```go
type Tool interface {
    Name() string
    Description() string
    Parameters() map[string]any
    Execute(ctx context.Context, args map[string]any) *ToolResult
}
```

Optional interface extensions:
- `ContextualTool` — adds `SetContext(channel, chatID string)` for channel-aware tools
- `AsyncTool` — adds `SetCallback(cb AsyncCallback)` for async/streaming tools

Return values use `ToolResult` constructors:

```go
tools.NewToolResult("success message")         // standard result
tools.ErrorResult("what went wrong").WithError(err) // error result
tools.SilentResult("internal note")            // not shown to user
tools.AsyncResult("processing...")             // async response
```

### Concurrency

- Use `sync.RWMutex` on all structs with shared mutable maps/slices
- Use `atomic.Bool` / `atomic.Int64` for simple flags and counters
- Use `sync.Map` for concurrent tracking maps where appropriate
- Prefer `sync.Once` for singleton initialization

### Logging

Use the structured logger from `pkg/logger`. Do not use `fmt.Println` or `log.Printf` directly
in production code paths:

```go
logger.InfoCF("component-name", "human readable message", map[string]any{
    "key": value,
    "count": n,
})
logger.WarnCF("component-name", "warning message", map[string]any{...})
logger.ErrorCF("component-name", "error message", map[string]any{"err": err})
logger.DebugCF("component-name", "debug message", map[string]any{...})
```

### Configuration

Config is loaded from `config.json` (JSON format). The canonical example is
`config/config.example.json`. Key sections:
- `agents.defaults` — model, workspace, token limits
- `model_list` — preferred model-centric format (replaces legacy `providers`)
- `channels` — per-channel adapter config
- `tools.*` — tool-specific config (web search, exec, cron, skills)

Use `pkg/config` for all config loading/access. Do not read config files directly elsewhere.

### Options Pattern

For optional configuration of types (especially providers), use the functional options pattern:

```go
type Option func(*Provider)

func WithRequestTimeout(d time.Duration) Option {
    return func(p *Provider) { p.timeout = d }
}

func NewProvider(apiKey string, opts ...Option) *Provider { ... }
```

---

## Testing Guidelines

- Use `github.com/stretchr/testify` for assertions (`assert`, `require`)
- Use table-driven tests for multiple scenarios:

```go
tests := []struct {
    name    string
    input   string
    want    string
    wantErr bool
}{
    {name: "empty input", input: "", want: "", wantErr: true},
}
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        got, err := MyFunc(tt.input)
        if tt.wantErr {
            require.Error(t, err)
            return
        }
        require.NoError(t, err)
        assert.Equal(t, tt.want, got)
    })
}
```

- Use `os.MkdirTemp("", "picoclaw-test-*")` + `defer os.RemoveAll(dir)` for temp dirs
- Test files live in the **same package** as the code under test (not `package foo_test`)
- Lint rules `funlen`, `maintidx`, `gocognit`, `gocyclo` are relaxed for `_test.go` files

---

## Pre-Commit Checklist

Before every commit, run `make check` and fix any errors it reports. Do not commit code that
fails `make check`.

---

## CI Pipeline

| Trigger | Jobs |
|---|---|
| Push to `main` | `make build-all` (cross-compile all platforms) |
| Pull request | `golangci-lint` + `go test ./...` (with `go generate ./...` first) |
| Manual tag dispatch | GoReleaser: multi-arch binaries + Docker images to GHCR & DockerHub |

CI always runs `go generate ./...` before lint and tests to ensure embedded assets are current.

---

## Project Layout

```
cmd/picoclaw/          CLI entry point and subcommands (cobra)
pkg/agent/             Core agent loop, context, registry, memory
pkg/channels/          Channel adapters (Telegram, Discord, Slack, LINE, etc.)
pkg/providers/         LLM provider implementations (Anthropic, OpenAI-compat)
pkg/tools/             Built-in tool implementations + Tool interface
pkg/config/            Config loading and migration
pkg/logger/            Structured logger (JSON to file, text to stdout)
pkg/session/           Session and conversation history manager
pkg/skills/            Skill registry, loader, installer
workspace/             Default agent workspace (AGENT.md, SOUL.md, skills/)
```
