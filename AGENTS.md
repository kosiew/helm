# Helm Repository Agent Guide

## Project summary
- This repository contains the Helm v4 CLI and supporting Go libraries. Most source lives under `cmd/`, `pkg/`, and `internal/`, with supporting scripts in `scripts/` and golden fixtures in `testdata/`.
- The module is `helm.sh/helm/v4` and targets Go 1.24.

## Coding guidelines
- **Go formatting:** run `make format` (wraps `goimports -local helm.sh/helm/v4`) on any Go files you touch. CI also enforces `gofmt`/`goimports` via `golangci-lint`.
- **License headers:** any new Go or shell script must start with the standard Apache 2.0 header used throughout the repo. You can verify with `make test-source-headers`.
- **Public API discipline:** packages under `pkg/` are part of the public SDK; avoid breaking exported APIs or behaviour. Prefer adding new symbols over changing or removing existing ones unless absolutely necessary. Use `internal/` for implementation details that can change freely.
- **Dependencies:** standard linters block deprecated packages like `github.com/pkg/errors`. Prefer the Go standard library `errors` helpers.
- **Tests:** follow existing patterns (table-driven tests, use `testing`, `require/assert` from `stretchr/testify` where already in use). Place golden fixtures in `testdata/` when needed.

## Required checks before committing
- Run Go unit tests for the areas you touched. The CI pipeline executes `make test-coverage` (per-package `go test` with coverage), so at minimum run `go test ./...` or `make test-coverage` locally.
- Ensure the binary still builds with `make build` when you change Go code.
- When you add or modify Go dependencies, run `go mod tidy` and ensure `go mod tidy -diff` reports no changes.
- If you modify Go or shell files, run `make test-source-headers` to confirm license headers are present.
- When you rely on `golangci-lint`, you can mirror CI by running `make test` (executes `golangci-lint run ./...` and the unit tests with the race detector).

## When touching documentation or metadata
- Markdown lives in the repo root (`README.md`, `CONTRIBUTING.md`, etc.). Keep line wrapping consistent with the surrounding text and prefer reference-style links already used in the document.
- Update user-facing docs or help text when CLI behaviour changes.

## Extra tips
- Acceptance tests in `../acceptance-testing` are optional and require an additional checkout; you can skip them unless explicitly asked.
- Use the provided scripts (for example `scripts/coverage.sh`) instead of rolling your own tooling when possible.
