# Implementation Plan: Migrate cc-permission-gate to tak848/ccgate

## 1. Design Decisions

### 1.1 Package Naming: `gate` (multiple sub-packages under internal/)

**Decision**: Use multiple focused packages under `internal/`.

- `internal/gate/` -- Core decision engine (DecidePermission, types, prompt construction)
- `internal/config/` -- Configuration loading, jsonnet evaluation, merge logic
- `internal/hookctx/` -- HookInput parsing, PermissionContext building, path extraction, transcript/settings loading
- `internal/gitutil/` -- Git command wrappers (extracted from hook_context.go)

**Rationale**: The monolithic `claudehooks` package mixed configuration, LLM calls, git operations, and input parsing. Splitting into focused packages improves testability (gitutil can be mocked), readability, and follows the single-responsibility principle. The name `gate` aligns with the project name and is idiomatic Go (short, descriptive).

### 1.2 Project Structure: Option B (cmd/ + multiple internal packages)

```
ccgate/
├── cmd/
│   └── cc-permission-gate/
│       └── main.go                 # Entry point: signal handling, logger init, stdin decode, stdout encode
├── internal/
│   ├── gate/
│   │   ├── gate.go                 # DecidePermission, PermissionDecision type
│   │   ├── gate_test.go
│   │   ├── anthropic.go            # callAnthropic, prompt building, schema generation
│   │   ├── anthropic_test.go
│   │   └── redact.go               # redactPromptInput, mustJSON
│   ├── config/
│   │   ├── config.go               # Config struct, DefaultConfig, LoadConfig, mergeConfigFile
│   │   ├── config_test.go
│   │   └── validate.go             # ValidateConfig (new)
│   ├── hookctx/
│   │   ├── input.go                # HookInput, HookToolInput, UnmarshalJSON, ToolInputText
│   │   ├── input_test.go
│   │   ├── context.go              # PermissionContext, BuildPermissionContext, referencedPaths
│   │   ├── context_test.go
│   │   ├── settings.go             # SettingsPermissions, LoadSettingsPermissions
│   │   ├── transcript.go           # RecentTranscript, LoadRecentTranscript, readTail
│   │   ├── transcript_test.go
│   │   ├── paths.go                # expandPaths, extractBashPaths, looksLikePathToken, uniqueNonEmpty
│   │   ├── paths_test.go
│   │   └── shell.go                # shellSplit
│   └── gitutil/
│       ├── gitutil.go              # Output, IsTracked (wrappers around git exec)
│       └── gitutil_test.go
├── version.go                      # var Version = "dev" (ldflags-injected)
├── .goreleaser.yml
├── .github/
│   └── workflows/
│       ├── ci.yml                  # Build + test on push/PR
│       └── release.yml             # Tag push -> goreleaser
├── Makefile
├── mise.toml
├── go.mod
├── go.sum
├── README.md
├── LICENSE
├── CLAUDE.md                       # Claude Code project instructions
├── example/
│   ├── permission-gate.jsonnet     # Example base config
│   └── permission-gate.schema.json # JSON Schema for config
└── z/
    └── 202603301558-migration-from-dotfiles.md
```

### 1.3 Release Automation: Option A (goreleaser + manual tag, like catatsuy/purl)

**Decision**: Start with the simpler approach. Manual `git tag v0.1.0 && git push --tags` triggers goreleaser.

**Rationale**: tagpr adds complexity. For a personal tool, manual tagging is sufficient. Can add tagpr later if needed.

### 1.4 Version Management

A root-level `version.go` with ldflags injection (like lambroll):

```go
package ccgate

var Version = "dev"
```

goreleaser and Makefile both inject via `-X github.com/tak848/ccgate.Version=v{{.Version}}`.

---

## 2. File-by-File Migration Plan with Quality Improvements

### 2.1 `cmd/cc-permission-gate/main.go`

**Source**: `dotfiles/go/cmd/cc-permission-gate/main.go`

**Current issues**:
- `_ = json.NewEncoder(os.Stdout).Encode(output)` silently discards encoding error
- `_ = os.MkdirAll(logDir, 0o755)` silently discards error
- `_ = os.Remove(prev)` and `_ = os.Rename(path, prev)` silently discard errors
- `atomicWriter` copies through an unnecessary `bytes.Buffer`
- No signal handling
- No version flag
- No graceful shutdown via context

**Improvements**:
1. **Signal handling**: Use `signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)` (lambroll pattern)
2. **`_main() int` pattern**: Return exit code from `_main()`, call `os.Exit()` in `main()` (lambroll pattern)
3. **Handle stdout encoding error**: Log and return non-zero exit code
4. **Remove atomicWriter buffer copy**: `O_APPEND` already provides atomicity for small writes; remove the unnecessary `bytes.Buffer` intermediate. Simplify to direct file write with mutex.
5. **Log rotation errors**: Log warnings instead of silently discarding
6. **Add `--version` flag**: Print version and exit
7. **Move output types to `internal/gate/`**: `permissionRequestResponse` and related types belong with the decision logic

**New structure**:
```go
package main

import (
    "context"
    "encoding/json"
    "flag"
    "fmt"
    "log/slog"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/tak848/ccgate"
    "github.com/tak848/ccgate/internal/config"
    "github.com/tak848/ccgate/internal/gate"
)

func main() { os.Exit(_main()) }

func _main() int {
    showVersion := flag.Bool("version", false, "print version and exit")
    flag.Parse()
    if *showVersion {
        fmt.Println("cc-permission-gate", ccgate.Version)
        return 0
    }

    logger, cleanup := initLogger()
    defer cleanup()
    slog.SetDefault(logger)

    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()

    var input hookctx.HookInput
    if err := json.NewDecoder(os.Stdin).Decode(&input); err != nil {
        slog.Error("failed to decode stdin", "error", err)
        return 1
    }
    // ... validation ...

    cfg, err := config.Load(input.Cwd)
    if err != nil {
        slog.Error("failed to load config", "error", err)
        return 1
    }

    start := time.Now()
    decision, ok, err := gate.DecidePermission(ctx, cfg, input)
    elapsed := time.Since(start)
    // ... logging, error handling ...

    if !ok {
        return 0 // fallthrough: no output
    }

    resp := gate.NewPermissionResponse(decision)
    if err := json.NewEncoder(os.Stdout).Encode(resp); err != nil {
        slog.Error("failed to encode response", "error", err)
        return 1
    }
    return 0
}
```

### 2.2 `internal/config/config.go`

**Source**: `dotfiles/go/internal/claudehooks/config.go`

**Current issues**:
- Inconsistent timeout defaults: `DefaultConfig()` has 20000ms, `LoadConfig()` fallback has 4000ms
- No validation after loading
- `mergeConfigFile` does an unnecessary `os.Stat` before `vm.EvaluateFile` (TOCTOU)
- `safeProjectLocalConfigPaths` silently skips on git errors (fail-open for security check)
- `projectLocalConfigPaths` and `safeProjectLocalConfigPaths` both call `gitOutput` for repo root (redundant)

**Improvements**:
1. **Unify timeout default**: Single constant `DefaultTimeoutMS = 20000`
2. **Remove redundant `os.Stat`**: Let `vm.EvaluateFile` handle the not-found case, catch `os.ErrNotExist` from the error
3. **Add `ValidateConfig`**: Validate provider name is supported, timeout is positive, model is non-empty
4. **Fix security concern in `safeProjectLocalConfigPaths`**: On git error, skip the file (fail-closed is correct, but log a warning)
5. **Extract git dependency**: Use `gitutil.Output` instead of inline `gitOutput`
6. **Single repo-root resolution**: Compute once and pass through

```go
// internal/config/config.go
package config

const (
    DefaultTimeoutMS  = 20_000
    DefaultModel      = "claude-haiku-4-5"
    DefaultProvider   = "anthropic"
    BaseConfigName    = "permission-gate.jsonnet"
    LocalConfigName   = "permission-gate.local.jsonnet"
)

type Config struct {
    Provider    ProviderConfig `json:"provider"`
    Allow       []string       `json:"allow"`
    Deny        []string       `json:"deny"`
    Environment []string       `json:"environment"`
}

type ProviderConfig struct {
    Name      string `json:"name"`
    Model     string `json:"model"`
    TimeoutMS int    `json:"timeout_ms"`
}

func Default() Config {
    return Config{
        Provider: ProviderConfig{
            Name:      DefaultProvider,
            Model:     DefaultModel,
            TimeoutMS: DefaultTimeoutMS,
        },
    }
}

// Validate checks Config invariants. Returns an error describing all violations.
func (c Config) Validate() error { ... }
```

### 2.3 `internal/config/validate.go` (NEW)

```go
package config

import (
    "errors"
    "fmt"
    "strings"
)

func (c Config) Validate() error {
    var errs []error
    if strings.TrimSpace(c.Provider.Name) == "" {
        errs = append(errs, fmt.Errorf("provider.name must not be empty"))
    }
    if strings.TrimSpace(c.Provider.Model) == "" {
        errs = append(errs, fmt.Errorf("provider.model must not be empty"))
    }
    if c.Provider.TimeoutMS <= 0 {
        errs = append(errs, fmt.Errorf("provider.timeout_ms must be positive, got %d", c.Provider.TimeoutMS))
    }
    return errors.Join(errs...)
}
```

### 2.4 `internal/gate/gate.go`

**Source**: `dotfiles/go/internal/claudehooks/anthropic.go` (DecidePermission function + types)

**Current issues**:
- `DecidePermission` returns `(PermissionDecision, bool, error)` -- the bool is awkward; could use a sentinel or a dedicated type
- Hardcoded default deny message duplicated in two places
- API key lookup mixed into decision logic

**Improvements**:
1. **Keep the `(decision, ok, error)` signature** -- it maps cleanly to the hook protocol (no output = fallthrough)
2. **Extract constants**: `DefaultDenyMessage`, `BehaviorAllow`, `BehaviorDeny`, `BehaviorFallthrough`
3. **Extract API key resolution** into a helper `resolveAPIKey() (string, error)`
4. **Response type**: Move `permissionRequestResponse` types from main.go into gate package as `PermissionResponse`
5. **Wrap all errors** with `fmt.Errorf("...: %w", err)`

```go
package gate

const (
    BehaviorAllow       = "allow"
    BehaviorDeny        = "deny"
    BehaviorFallthrough = "fallthrough"
    DefaultDenyMessage  = "危険な可能性が高いため、自動許可しません。"
)

type PermissionDecision struct {
    Behavior string `json:"behavior"`
    Message  string `json:"message,omitempty"`
}

// PermissionResponse is the JSON structure written to stdout for Claude Code.
type PermissionResponse struct {
    HookSpecificOutput hookSpecificOutput `json:"hookSpecificOutput"`
}

func NewPermissionResponse(d PermissionDecision) PermissionResponse { ... }

func DecidePermission(ctx context.Context, cfg config.Config, input hookctx.HookInput) (PermissionDecision, bool, error) { ... }
```

### 2.5 `internal/gate/anthropic.go`

**Source**: `dotfiles/go/internal/claudehooks/anthropic.go` (callAnthropic, prompt building, schema)

**Current issues**:
- Redundant timeout fallback (`if timeout <= 0 { timeout = 20 * time.Second }`) -- should never happen after config validation
- `MaxTokens: 4096` hardcoded magic number
- `option.WithMaxRetries(5)` hardcoded magic number
- `permissionOutputSchema` silently falls back to empty schema on marshal error
- `mustJSON` silently returns `"{}"` on error

**Improvements**:
1. **Named constants**: `maxTokens = 4096`, `maxRetries = 5`
2. **Remove redundant timeout guard**: Config validation guarantees positive timeout
3. **Schema generation errors**: Return error from schema generation, fail fast
4. **`mustJSON` -> return error**: Change to `marshalJSON(v any) (string, error)` where callers handle the error. Keep a `mustJSON` that panics for truly impossible cases (test helpers only).
5. **Extract prompt building** into `prompt.go` or keep in `anthropic.go` (keep together since tightly coupled)

### 2.6 `internal/hookctx/input.go`

**Source**: `dotfiles/go/internal/claudehooks/hook_context.go` (HookInput, HookToolInput, UnmarshalJSON, ToolInputText)

**Current issues**:
- `ToolInputText()` concatenates everything without separators meaningful to the consumer
- No validation of HookInput fields

**Improvements**:
1. **Add `Validate()` method to HookInput**: Check ToolName is non-empty, HookEventName is "PermissionRequest"
2. **Keep ToolInputText as-is**: It provides a reasonable text representation for path extraction

### 2.7 `internal/hookctx/context.go`

**Source**: `dotfiles/go/internal/claudehooks/hook_context.go` (BuildPermissionContext)

**Current issues**:
- Multiple sequential `gitOutput` calls that could be batched or cached
- `IsWorktree` detection via string matching on `.git/worktrees/` is fragile

**Improvements**:
1. **Use `gitutil` package**: Replace inline git calls with `gitutil.Output`
2. **Worktree detection**: Compare `--git-dir` and `--git-common-dir` (they differ in worktrees). This is more robust than string matching.
3. **Cache repo root**: Compute once, pass to sub-functions

### 2.8 `internal/hookctx/paths.go`

**Source**: `dotfiles/go/internal/claudehooks/hook_context.go` (referencedPaths, expandPaths, extractBashPaths, looksLikePathToken, uniqueNonEmpty)

**Current issues**:
- O(n^2) `uniqueNonEmpty` using `slices.Contains` on output slice
- `shellSplit` is a fragile hand-rolled parser (but adequate for the use case)

**Improvements**:
1. **Use map for dedup**: Replace O(n^2) with O(n) using `seen` map
2. **Keep shellSplit**: It handles the common cases well enough. Add more test cases for edge cases.

```go
func uniqueNonEmpty(values []string) []string {
    seen := make(map[string]struct{}, len(values))
    out := make([]string, 0, len(values))
    for _, v := range values {
        if v == "" {
            continue
        }
        if _, ok := seen[v]; ok {
            continue
        }
        seen[v] = struct{}{}
        out = append(out, v)
    }
    return out
}
```

### 2.9 `internal/hookctx/transcript.go`

**Source**: `dotfiles/go/internal/claudehooks/hook_context.go` (LoadRecentTranscript, readTail)

**Current issues**:
- Silent error swallowing in `LoadRecentTranscript` (returns empty on all errors)
- Magic numbers: `tailBytes = 64 * 1024`, `maxUserMessages = 3`, `maxToolCalls = 5`

**Improvements**:
1. **Return error**: Change signature to `LoadRecentTranscript(path string) (RecentTranscript, error)`. Callers can log and proceed with empty.
2. **Named constants remain**: They are already named, which is fine. Keep them.

### 2.10 `internal/hookctx/settings.go`

**Source**: `dotfiles/go/internal/claudehooks/hook_context.go` (LoadSettingsPermissions)

**Current issues**:
- Silently ignores all errors

**Improvements**:
1. **Return error for non-ENOENT failures**: File-not-found is expected; JSON parse errors should be reported
2. **Use `gitutil.Output`** for repo root detection

### 2.11 `internal/gitutil/gitutil.go` (NEW)

Extract git operations into a dedicated package for testability and reuse.

```go
package gitutil

import (
    "errors"
    "fmt"
    "os/exec"
    "strings"
)

// Output runs `git <args...>` in the given directory and returns trimmed stdout.
func Output(dir string, args ...string) (string, error) {
    cmd := exec.Command("git", args...)
    if dir != "" {
        cmd.Dir = dir
    }
    out, err := cmd.Output()
    if err != nil {
        return "", fmt.Errorf("git %s: %w", strings.Join(args, " "), err)
    }
    return strings.TrimSpace(string(out)), nil
}

// IsTracked returns true if the file at path is tracked by git in the given repo root.
// Returns false if the file does not exist or the directory is not a git repo.
func IsTracked(repoRoot, path string) (bool, error) {
    // ... improved version of isTrackedProjectFile ...
}

// RepoRoot returns the top-level directory of the git repository containing dir.
func RepoRoot(dir string) (string, error) {
    return Output(dir, "rev-parse", "--show-toplevel")
}
```

### 2.12 `version.go` (NEW, root package)

```go
package ccgate

// Version is set by ldflags at build time.
var Version = "dev"
```

---

## 3. Release Automation

### 3.1 `.goreleaser.yml`

Following the catatsuy/purl pattern with version 2 syntax from lambroll:

```yaml
version: 2
project_name: cc-permission-gate
before:
  hooks:
    - go mod tidy
builds:
  - main: ./cmd/cc-permission-gate
    binary: cc-permission-gate
    ldflags:
      - -s -w
      - -X github.com/tak848/ccgate.Version=v{{.Version}}
    env:
      - CGO_ENABLED=0
    goos:
      - darwin
      - linux
    goarch:
      - amd64
      - arm64
archives:
  - name_template: '{{ .ProjectName }}-{{ .Os }}-{{ .Arch }}'
release:
  prerelease: auto
checksum:
  name_template: checksums.txt
changelog:
  sort: asc
  filters:
    exclude:
      - '^docs:'
      - '^test:'
      - '^ci:'
```

### 3.2 `.github/workflows/release.yml`

```yaml
name: release
on:
  push:
    tags:
      - "v[0-9]+.[0-9]+.[0-9]+"
permissions:
  contents: write
jobs:
  goreleaser:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false
      - uses: actions/setup-go@v5
        with:
          go-version: "1.25"
      - uses: goreleaser/goreleaser-action@v6
        with:
          version: '~> v2'
          args: release --clean
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 3.3 `.github/workflows/ci.yml`

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
permissions:
  contents: read
jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false
      - uses: actions/setup-go@v5
        with:
          go-version: "1.25"
      - run: make vet
      - run: make staticcheck
      - run: make test
```

---

## 4. Makefile

```makefile
.PHONY: all build test vet staticcheck clean install

BINARY := cc-permission-gate
VERSION := $(shell git describe --tags --always --dirty 2>/dev/null || echo "dev")
LDFLAGS := -s -w -X github.com/tak848/ccgate.Version=$(VERSION)

all: build

build:
	go build -ldflags "$(LDFLAGS)" -o bin/$(BINARY) ./cmd/cc-permission-gate

install: build
	install -d $(HOME)/.claude/bin
	install bin/$(BINARY) $(HOME)/.claude/bin/$(BINARY)

vet:
	go vet ./...

staticcheck:
	staticcheck -checks="all,-ST1000" ./...

test:
	go test -race -cover -v ./...

cover:
	go test -coverprofile=coverage.out ./...
	go tool cover -html=coverage.out

clean:
	rm -rf bin/
```

---

## 5. mise.toml

```toml
[tools]
go = "1.25"

[tools.staticcheck]
version = "latest"
```

Note: The user specified Go 1.25 ("one version before latest" where latest is 1.26). mise handles Go installation. staticcheck is also managed via mise.

---

## 6. go.mod

```
module github.com/tak848/ccgate

go 1.25

require (
    github.com/anthropics/anthropic-sdk-go v1.27.1
    github.com/google/go-jsonnet v0.22.0
    github.com/invopop/jsonschema v0.13.0
)
```

---

## 7. Step-by-Step Implementation Order

### Phase 1: Project Skeleton (PR #1)

1. Create `go.mod` with `module github.com/tak848/ccgate` and `go 1.25`
2. Create `version.go` at project root
3. Create `mise.toml`
4. Create `Makefile`
5. Create `.goreleaser.yml`
6. Create `.github/workflows/ci.yml`
7. Create `.github/workflows/release.yml`
8. Create `README.md` with basic description and installation instructions
9. Create `LICENSE` (MIT or whatever the user prefers)
10. Create `CLAUDE.md` with project conventions
11. Run `go mod tidy` to verify

### Phase 2: Core Internal Packages (PR #2)

1. Create `internal/gitutil/gitutil.go` -- extracted and improved git wrappers
2. Create `internal/gitutil/gitutil_test.go`
3. Create `internal/config/config.go` -- Config types, Default(), Load(), mergeConfigFile()
4. Create `internal/config/validate.go` -- Validate()
5. Create `internal/config/config_test.go` -- migrate and expand existing tests
6. Run `go mod tidy` and `make test`

### Phase 3: Hook Context Package (PR #3)

1. Create `internal/hookctx/input.go` -- HookInput, HookToolInput, UnmarshalJSON
2. Create `internal/hookctx/input_test.go`
3. Create `internal/hookctx/context.go` -- PermissionContext, BuildPermissionContext
4. Create `internal/hookctx/context_test.go`
5. Create `internal/hookctx/paths.go` -- path extraction with O(n) dedup
6. Create `internal/hookctx/paths_test.go` -- expand existing shellSplit and extractBashPaths tests
7. Create `internal/hookctx/shell.go` -- shellSplit (unchanged, well-tested)
8. Create `internal/hookctx/transcript.go` -- LoadRecentTranscript with error return
9. Create `internal/hookctx/transcript_test.go`
10. Create `internal/hookctx/settings.go` -- LoadSettingsPermissions with error return
11. Run `make test`

### Phase 4: Gate Package (PR #4)

1. Create `internal/gate/gate.go` -- DecidePermission, PermissionDecision, PermissionResponse, constants
2. Create `internal/gate/gate_test.go`
3. Create `internal/gate/anthropic.go` -- callAnthropic, prompt construction, schema generation
4. Create `internal/gate/anthropic_test.go`
5. Create `internal/gate/redact.go` -- redactPromptInput, marshalJSON
6. Run `make test`

### Phase 5: Entry Point (PR #5)

1. Create `cmd/cc-permission-gate/main.go` -- signal handling, logger, stdin/stdout, version flag
2. Create `example/permission-gate.jsonnet`
3. Create `example/permission-gate.schema.json`
4. Run full integration: `echo '{"tool_name":"Bash",...}' | go run ./cmd/cc-permission-gate/`
5. Run `make build && make install`
6. Verify `make vet && make staticcheck && make test`

### Phase 6: Polish (PR #6)

1. Add comprehensive README with usage, configuration, and installation
2. Review and finalize CLAUDE.md
3. Tag `v0.1.0` and verify goreleaser works

---

## 8. Key Code Improvements Summary

| Area | Before | After |
|------|--------|-------|
| Error handling | `_ = json.NewEncoder(os.Stdout).Encode(output)` | Return error, log, exit non-zero |
| Error handling | `_ = os.MkdirAll(...)`, `_ = os.Remove(...)` | Log warnings for non-critical, return errors for critical |
| Timeout defaults | 4000ms fallback in LoadConfig, 20000ms in DefaultConfig, 20s in callAnthropic | Single constant `DefaultTimeoutMS = 20_000`, validation ensures positive |
| Dedup algorithm | O(n^2) `slices.Contains` on output slice | O(n) map-based dedup |
| Signal handling | None | `signal.NotifyContext` with SIGINT/SIGTERM |
| Exit codes | `return` (always 0) | `_main() int` pattern with meaningful exit codes |
| Package structure | Monolithic `claudehooks` | `config`, `gate`, `hookctx`, `gitutil` |
| Version info | None | `--version` flag, ldflags injection |
| Config validation | None | `Config.Validate()` after load |
| Worktree detection | String matching `.git/worktrees/` | Compare `--git-dir` vs `--git-common-dir` |
| atomicWriter | Unnecessary bytes.Buffer copy | Direct mutex-guarded write (O_APPEND handles kernel atomicity) |
| Schema errors | Silent fallback to `{"type":"object"}` | Return error, fail fast |
| Git error wrapping | Raw `cmd.Output()` errors | `fmt.Errorf("git %s: %w", args, err)` |
| Transcript loading | Silently returns empty | Returns `(RecentTranscript, error)` |
| Settings loading | Silently ignores parse errors | Reports non-ENOENT errors |
| mustJSON | Returns `"{}"` on error (silent) | `marshalJSON` returns error; `mustJSON` only for tests |
| Magic numbers | `4096`, `5`, `64*1024` inline | Named constants with documentation |
| Input validation | None | `HookInput.Validate()` checks ToolName, HookEventName |

---

## 9. Testing Strategy

### Unit Tests (every package)
- **Table-driven tests** for all pure functions (shellSplit, expandPaths, extractBashPaths, looksLikePathToken, uniqueNonEmpty, truncate, Validate)
- **Error path tests**: Invalid JSON input, missing config files, git errors, API failures
- **Config merge tests**: Base only, base + local, override semantics, empty overrides

### Integration Tests
- **Config loading**: Real jsonnet files in testdata/
- **Git operations**: Use `t.TempDir()` + `git init` (existing pattern from config_test.go)
- **HookInput unmarshaling**: Real JSON payloads in testdata/

### What NOT to test (initially)
- Anthropic API calls (requires API key, network). Can add later with httptest mock.
- Full end-to-end stdin/stdout flow (manual testing + CI build verification).

---

## 10. Files NOT Migrated

These stay in dotfiles:
- `dot_claude/permission-gate.jsonnet` -- User's actual config (dotfiles-managed)
- `dot_claude/permission-gate.schema.json` -- Deployed alongside config
- `dot_claude/settings.jsonnet` -- Claude Code settings (unrelated)
- `dot_claude/permission-rules.libsonnet` -- PreToolUse rules (separate hook)
- Build scripts (chezmoi-specific)

The ccgate repo includes these as `example/` files for reference only.
