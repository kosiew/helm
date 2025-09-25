# Helm Repository Agent Guide

## Project summary

* This repository contains the Helm v4 CLI and supporting Go libraries. Most source lives under `cmd/`, `pkg/`, and `internal/`, with supporting scripts in `scripts/` and golden fixtures in `testdata/`.
* The module is `helm.sh/helm/v4` and targets Go 1.24.

## Coding guidelines

* **Go formatting:** run `make format` (wraps `goimports -local helm.sh/helm/v4`) on any Go files you touch. CI also enforces `gofmt`/`goimports` via `golangci-lint`.
* **License headers:** any new Go or shell script must start with the standard Apache 2.0 header used throughout the repo. You can verify with `make test-source-headers`.
* **Public API discipline:** packages under `pkg/` are part of the public SDK; avoid breaking exported APIs or behaviour. Prefer adding new symbols over changing or removing existing ones unless absolutely necessary. Use `internal/` for implementation details that can change freely.
* **Dependencies:** standard linters block deprecated packages like `github.com/pkg/errors`. Prefer the Go standard library `errors` helpers. Prefer maintained, well-reviewed forks where upstream is deprecated (for example, `ProtonMail/go-crypto` instead of `golang.org/x/crypto/openpgp`).
* **Tests:** follow existing patterns (table-driven tests, use `testing`, `require/assert` from `stretchr/testify` where already in use). Place golden fixtures in `testdata/` when needed.
* **Refactoring style:**

  * Prefer declarative helpers such as `slices.Contains` / `slices.ContainsFunc` over manual flag variables and nested loops when checking membership/conditions in slices.
  * Prefer `testify/assert` (or `require`) for concise test assertions instead of ad-hoc `if` checks and `t.Fatalf`.
  * See **Refactoring guidelines** below for additional patterns and concrete examples.

## Refactoring guidelines

To keep the codebase idiomatic, safe, and maintainable, prefer the following refactor patterns. These are guidelines, not absolute rules — use judgement.

### 1) Prefer declarative collection helpers over manual loops

**Why:** reduces boilerplate, communicates intent ("any element matches this predicate"), and avoids mutable flags.

**Before (imperative loops + flags):**

```go
hasEdDSA := false
for _, entity := range signer.KeyRing {
    if entity.PrimaryKey != nil && entity.PrimaryKey.PubKeyAlgo == packet.PubKeyAlgoEdDSA {
        hasEdDSA = true
        break
    }
    for _, subkey := range entity.Subkeys {
        if subkey.PublicKey != nil && subkey.PublicKey.PubKeyAlgo == packet.PubKeyAlgoEdDSA {
            hasEdDSA = true
            break
        }
    }
    if hasEdDSA {
        break
    }
}
```

**After (declarative):**

```go
hasEdDSA := slices.ContainsFunc(signer.KeyRing, func(entity Entity) bool {
    if entity.PrimaryKey != nil && entity.PrimaryKey.PubKeyAlgo == packet.PubKeyAlgoEdDSA {
        return true
    }
    return slices.ContainsFunc(entity.Subkeys, func(subkey SubKey) bool {
        return subkey.PublicKey != nil && subkey.PublicKey.PubKeyAlgo == packet.PubKeyAlgoEdDSA
    })
})
```

> Note: `slices.ContainsFunc` is available in the standard library (Go 1.21+). If unavailable in older toolchains, consider a small local helper `any(s []T, pred func(T) bool) bool`.

### 2) Prefer `testify/assert` / `require` in tests

**Why:** clearer failure messages, less boilerplate, improves readability.

**Before:**

```go
if !hasEdDSA {
    t.Fatalf("expected %s to include an Ed25519 public key", testMixedKeyring)
}
if signer.Entity == nil {
    t.Fatalf("signer entity was nil")
}
```

**After:**

```go
assert.True(t, hasEdDSA)
assert.NotNil(t, signer.Entity)
```

When the test cannot continue after an assertion, use `require.*`.

### 3) Use early returns / guard clauses to reduce nesting

**Why:** Flatter code is easier to read and reason about.

**Before:**

```go
if entity != nil {
    if entity.PrimaryKey != nil {
        if entity.PrimaryKey.PubKeyAlgo == packet.PubKeyAlgoEdDSA {
            return true
        }
    }
}
return false
```

**After:**

```go
if entity == nil || entity.PrimaryKey == nil {
    return false
}
return entity.PrimaryKey.PubKeyAlgo == packet.PubKeyAlgoEdDSA
```

### 4) Extract well-named predicate/helper functions

**Why:** Reusing a short, descriptive function improves intent and reduces duplicated condition logic.

```go
func isEdDSAKey(pk *Key) bool {
    return pk != nil && pk.PubKeyAlgo == packet.PubKeyAlgoEdDSA
}
```

Then use it in higher-level checks and in tests.

### 5) Replace mutable flags with pure computations

**Why:** Pure functions are easier to test and reason about than code with cross-scope mutations.

**Before:**

```go
found := false
for ... {
    if cond { found = true }
}
if found { ... }
```

**After:**

```go
func anyMatch[T any](slice []T, pred func(T) bool) bool {
    for _, v := range slice {
        if pred(v) { return true }
    }
    return false
}

if anyMatch(signer.KeyRing, hasEdDSAEntity) { ... }
```

### 6) Table-driven tests

**Why:** Easier to add cases, clearer surface area of behaviour, and consistent test patterns.

```go
tests := []struct{
    name string
    pk   *Key
    want bool
}{
    {"nil", nil, false},
    {"rsa", &Key{PubKeyAlgo: packet.PubKeyAlgoRSA}, false},
    {"eddsa", &Key{PubKeyAlgo: packet.PubKeyAlgoEdDSA}, true},
}
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        assert.Equal(t, tt.want, isEdDSAKey(tt.pk))
    })
}
```

### 7) Reduce duplication with interfaces / composition

If `Entity` and `SubKey` expose similar behaviour (e.g., `GetPublicKey()`), define a small interface and write generic helpers.

```go
type KeyHolder interface {
    PublicKey() *Key
}

func hasEdDSAOn(holder KeyHolder) bool {
    return isEdDSAKey(holder.PublicKey())
}
```

### 8) Modern error handling

Prefer `errors.Is` / `errors.As` / `errors.Join` and wrap context with `fmt.Errorf("...: %w", err)` where appropriate. Avoid silent failures.

### 9) Context propagation

Functions performing I/O, crypto operations, or any cancellable work should accept `context.Context` to support timeouts and cancellation.

### 10) Small, targeted refactors

Prefer many small refactors that are easy to review and revert rather than large sweeping changes. Each PR should be reviewable on its own merits.

## Required checks before committing

* Run Go unit tests for the areas you touched. The CI pipeline executes `make test-coverage` (per-package `go test` with coverage), so at minimum run `go test ./...` or `make test-coverage` locally.
* Ensure the binary still builds with `make build` when you change Go code.
* When you add or modify Go dependencies, run `go mod tidy` and ensure `go mod tidy -diff` reports no changes.
* If you modify Go or shell files, run `make test-source-headers` to confirm license headers are present.
* When you rely on `golangci-lint`, you can mirror CI by running `make test` (executes `golangci-lint run ./...` and the unit tests with the race detector).

## When touching documentation or metadata

* Markdown lives in the repo root (`README.md`, `CONTRIBUTING.md`, etc.). Keep line wrapping consistent with the surrounding text and prefer reference-style links already used in the document.
* Update user-facing docs or help text when CLI behaviour changes.

## Extra tips

* Acceptance tests in `../acceptance-testing` are optional and require an additional checkout; you can skip them unless explicitly asked.
* Use the provided scripts (for example `scripts/coverage.sh`) instead of rolling your own tooling when possible.