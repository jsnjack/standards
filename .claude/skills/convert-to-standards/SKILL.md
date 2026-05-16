---
name: convert-to-standards
description: 'Convert a Go project to jsnjack/standards conventions. Use when onboarding a repo, adding AGENTS.md, scaffolding CLAUDE.md, updating Makefile ldflags, or applying universal and Go-specific coding standards to a project.'
argument-hint: 'path to the project/repo to convert'
---

# Convert Go project to standards

Read `AGENTS.universal.md` first — it defines all conventions referenced below.

Execute each step, audit current state, fix what's missing, report what changed.

---

## Step 1 — Add AGENTS.universal.md and AGENTS.go.md

```bash
curl -sL https://raw.githubusercontent.com/jsnjack/standards/master/AGENTS.universal.md \
    -o AGENTS.universal.md
curl -sL https://raw.githubusercontent.com/jsnjack/standards/master/AGENTS.go.md \
    -o AGENTS.go.md
```

---

## Step 1b — Add CLAUDE.md

Create `CLAUDE.md` in the repo root with a single line so Claude Code picks up
the same instructions as every other agent:

```
@AGENTS.md
```

---

## Step 2 — Create or update AGENTS.md

Structure:
```
# AGENTS.md

> See [AGENTS.universal.md](./AGENTS.universal.md) and [AGENTS.go.md](./AGENTS.go.md) for universal conventions.
> Refresh: `make standards`

---

## Overview
<one paragraph: what it does and who uses it>

---

## Architecture
<directory tree with one-line descriptions per file/package>

---

## Key Flows             ← omit if the project is simple
<numbered list of the main request/data paths through the code>

---

## Build & Run
<project-specific commands and smoke tests>

---

## Configuration         ← omit if there is no non-trivial config
<where config lives, format, and key fields an agent needs to know>

---

## Design Decisions
<bullets — non-obvious choices only, one line rationale each>

---

## Gotchas               ← omit if none
<bullets — things that have caused bugs or confusion before>

---

## Known Issues          ← omit if none
<bullets — confirmed bugs or limitations; include workaround if one exists>
```

---

## Step 3 — Update Makefile

Ensure the main/cmd package exposes a version variable that ldflags can stamp:
```go
// Version is set at build time via ldflags; defaults to "dev".
var Version = "dev"
```

The `LDFLAGS` line in the Makefile must reference it as `-X <import-path>.Version`.

Ensure `.gitignore` contains:
```
bin/
<binary-name>
.monova*
```
The root `<binary-name>` entry ignores the symlink created by `make build`.

Add or fix all standard targets. Preserve project-specific targets unchanged.

Standard targets template:
```makefile
BINARY  := <name>
PKG     := ./...
VERSION := 0.0.0
MONOVA  := $(shell which monova 2> /dev/null)
LDFLAGS  = -ldflags="-X <pkg>.Version=$(VERSION)"

export PATH := $(PATH):$(shell go env GOPATH)/bin

version:
ifdef MONOVA
override VERSION = $(shell monova)
override LDFLAGS = -ldflags="-X <pkg>.Version=$(VERSION)"
else
$(info "Install monova with: grm install jsnjack/monova")
endif

test:
go test $(PKG)

vet:
go vet $(PKG)

fmt:
@command -v goimports >/dev/null 2>&1 || { \
  echo "goimports is not installed. Install it with:"; \
  echo "  go install golang.org/x/tools/cmd/goimports@latest"; \
  exit 1; \
}
goimports -w .

lint: vet
@command -v golangci-lint >/dev/null 2>&1 || { \
  echo "golangci-lint is not installed. Install it with:"; \
  echo "  grm install golangci/golangci-lint"; \
  exit 1; \
}
golangci-lint run

check: fmt vet build test lint
@echo "==> make check: all green"

standards:
curl -sL https://raw.githubusercontent.com/jsnjack/standards/master/AGENTS.universal.md \
    -o AGENTS.universal.md
curl -sL https://raw.githubusercontent.com/jsnjack/standards/master/AGENTS.go.md \
    -o AGENTS.go.md

bin/$(BINARY): bin/$(BINARY)_linux_amd64
	cp $$< $$@
	ln -sf bin/$(BINARY) $(BINARY)
bin/$(BINARY)_linux_amd64: version
	GOOS=linux GOARCH=amd64 go build $(LDFLAGS) -o $$@
bin/$(BINARY)_linux_arm64: version
	GOOS=linux GOARCH=arm64 go build $(LDFLAGS) -o $$@
bin/$(BINARY)_darwin_amd64: version
	GOOS=darwin GOARCH=amd64 go build $(LDFLAGS) -o $$@
bin/$(BINARY)_darwin_arm64: version
	GOOS=darwin GOARCH=arm64 go build $(LDFLAGS) -o $$@

build: bin/$(BINARY) bin/$(BINARY)_linux_amd64 bin/$(BINARY)_linux_arm64 bin/$(BINARY)_darwin_amd64 bin/$(BINARY)_darwin_arm64

release: build
	tar -czf bin/$(BINARY)_linux_amd64.tar.gz  --transform 's|.*/$(BINARY)_.*|$(BINARY)|' bin/$(BINARY)_linux_amd64
	tar -czf bin/$(BINARY)_linux_arm64.tar.gz  --transform 's|.*/$(BINARY)_.*|$(BINARY)|' bin/$(BINARY)_linux_arm64
	tar -czf bin/$(BINARY)_darwin_amd64.tar.gz --transform 's|.*/$(BINARY)_.*|$(BINARY)|' bin/$(BINARY)_darwin_amd64
	tar -czf bin/$(BINARY)_darwin_arm64.tar.gz --transform 's|.*/$(BINARY)_.*|$(BINARY)|' bin/$(BINARY)_darwin_arm64
	grm release jsnjack/$(BINARY) \
		-f bin/$(BINARY)_linux_amd64.tar.gz \
		-f bin/$(BINARY)_linux_arm64.tar.gz \
		-f bin/$(BINARY)_darwin_amd64.tar.gz \
		-f bin/$(BINARY)_darwin_arm64.tar.gz \
		-t "v`monova`"

clean:
rm -rf bin/ $(BINARY)

.PHONY: version build release test vet fmt lint check standards clean
```

---

## Step 4 — Implement logging

Follow logging conventions in `AGENTS.universal.md`. Add `logger.go`:

```go
package main // or cmd

import (
    "io"
    "log/slog"
    "os"
)

const LevelTrace = slog.Level(-8)

var L *slog.Logger

func initLogger(tracePath, level string) func() {
    var w io.Writer = io.Discard
    cleanup := func() {}
    if tracePath != "" {
        f, err := os.OpenFile(tracePath, os.O_WRONLY|os.O_CREATE|os.O_TRUNC, 0o600)
        if err == nil {
            w = f
            cleanup = func() { _ = f.Close() }
        }
    }
    lvl := slog.LevelWarn
    switch level {
    case "debug":
        lvl = slog.LevelDebug
    case "trace":
        lvl = LevelTrace
    }
    h := slog.NewTextHandler(w, &slog.HandlerOptions{Level: lvl})
    L = slog.New(h)
    slog.SetDefault(L)
    return cleanup
}
```

Add flags to every command:
```go
flags.BoolVarP(&debug, "debug", "d", false,
    "Debug-level logging on stderr.")
flags.BoolVar(&trace, "trace", false,
    "Trace-level logs to /tmp/<binary>.log (truncated each run).")
```

Wire in `RunE`:
```go
level, tracePath := "", ""
switch {
case trace:
    level, tracePath = "trace", "/tmp/<binary>.log"
case debug:
    level = "debug"
}
cleanup := initLogger(tracePath, level)
defer cleanup()
```

Replace all `log.Printf` / `log.Println` with `slog` equivalents using
structured fields (`slog.String`, `slog.Int`, etc.).

---

## Step 5 — Fix error handling

Comply with error conventions in `AGENTS.universal.md`:
- `fmt.Errorf("context: %w", err)` everywhere.
- `http.Error` + immediate return in handlers.
- No `log.Fatal` / `os.Exit` in goroutines.

---

## Step 6 — Add missing tests

Comply with test conventions in `AGENTS.universal.md`.
Write table-driven tests for all untested exported functions.

---

## Step 7 — Run make check

```bash
make check
```

All targets must pass before committing.

