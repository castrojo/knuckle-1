# Quality Agent Patterns and Coverage Gotchas

Load when: working on coverage PRs, reviewing quality agent issues, or updating `just cover-check`.

## Coverage gate baseline (2026-06-01, post PR #675)

All 16 packages are in `just cover-check`. Authoritative source: `Justfile :: cover-check` and `docs/CI-AND-TESTING.md`.

| Package | Gate |
|---|---|
| `internal/*` (model, iso, runner, demo, validate, probe, install, ignition, bakery) | 100% |
| `scripts/catalog_check` | 100% |
| `cmd/compile-butane-fresh` | 100% |
| `internal/github` | 96% |
| `internal/wizard` | 99% |
| `internal/headless` | 99% |
| `internal/tui` | 99% |
| `cmd/knuckle` | 85% |

## Infeasible coverage — do not chase

**`internal/tui` remaining 0.3%:**
- `tea.Run()` at `tui.go:1264` — requires a real PTY. Not testable without dependency injection of the program runner.
- `os.UserHomeDir()` failure at `tui.go:1279` — requires a broken `/etc/passwd`. Not achievable in a normal test environment.
- Both are guarded by the `programRunner` injection point and the `if err != nil { return nil }` nil-guard. Leave gate at 99%.

**`cmd/knuckle` remaining ~14%:**
- TTY-gated TUI launch path in `main()`. Requires a real PTY. Gate is 85% — all non-TTY branches are covered.

## Quality agent stale coverage — recurring pattern

**Symptom:** Quality agent files a coverage PR for a function that is already at 100%.

**Cause:** The agent snapshots coverage at issue-creation time. By the time the PR is written, another PR may have already covered the function.

**Fix before touching any coverage issue:**
```bash
go test -count=1 -coverprofile=/tmp/check.cov ./internal/<pkg>/...
go tool cover -func=/tmp/check.cov | grep -v "100.0%"
```
If the target function already shows 100%, close the issue as stale — do not open a PR.

**Affected issues (resolved, kept for pattern reference):** #616 (splitSSHKeys/mergeKeys already 100% on main before PR was opened).

## Quality agent PRs — pre-approve checklist

Before approving any quality agent PR, check all three:

1. **gofmt** — new test files must use tab indentation: `gofmt -l <file>` must return empty.
2. **`t.Fatal` missing `return`** (SA5011) — add `return` on the line after every `t.Fatal` that guards a subsequent dereference.
3. **`t.Error` before dereference** — `t.Error` does not stop the test. Use `t.Fatal` + `return` instead.

```bash
gofmt -l <file>                    # must be empty
grep -n "t\.Error" <file>          # check each: dereference after?
grep -A2 "t\.Fatal" <file>         # check each: return present?
```

### SA5011 pattern — always add `return` after `t.Fatal`

`golangci-lint-action` in CI catches SA5011 (nil-deref after t.Fatal) even when `golangci-lint run ./...` locally reports clean. **Always add `return` immediately after every `t.Fatal` nil-check guard:**

```go
// WRONG — SA5011: golangci-lint catches this even when local lint passes
if result == nil {
    t.Fatal("expected non-nil result")
}
result.Field  // potential nil deref

// CORRECT
if result == nil {
    t.Fatal("expected non-nil result")
    return  // ← REQUIRED even though t.Fatal stops the test logically
}
result.Field
```

Failing to do this blocks the entire merge queue for all open PRs.

### `t.Error` vs `t.Fatal` before dereference

`t.Error` does not stop the test — it only marks it failed. Using `t.Error` before a field access panics if the value is nil:

```go
// WRONG — panics if m.err == nil
if m.err == nil {
    t.Error("expected error")
}
if !strings.Contains(m.err.Error(), "...") { // panics

// CORRECT
if m.err == nil {
    t.Fatal("expected error")
    return
}
if !strings.Contains(m.err.Error(), "...") {
```

### detectLocalSSHKeys: `UserHomeDir` error path is testable

`os.UserHomeDir()` returns an error when `HOME=""` on Linux:

```go
t.Setenv("HOME", "")
keys := detectLocalSSHKeys()  // triggers error path — returns nil
```

Use `t.Setenv` (auto-restored after test) rather than `os.Setenv` to avoid test pollution.

## BATS test / script alignment — three rules

When modifying `scripts/qa-test-pr.sh`, the BATS test suite greps the mock git log for literal strings. Follow these rules exactly:

1. **`remove_worktree_path()` must unconditionally call `git worktree remove --force`** when the path exists on disk (not just when registered in `git worktree list`):
   ```bash
   if [[ -e "$path" ]]; then
     git worktree remove --force "$path" 2>/dev/null || true
     if [[ -e "$path" ]]; then rm -rf "$path"; fi
   fi
   ```
2. **`--force` before path**: `git worktree remove --force "$path"` (not `"$path" --force`)
3. **Use `git branch -D "$ref"` directly** for local branch cleanup — not `git update-ref -d || git branch -D` (the mock makes `update-ref` succeed, so the fallback is never reached)

### Merged tests that break main

When a test-first PR merges (tests for behavior not yet implemented), all subsequent PRs will fail BATS in CI. **Always run `bats scripts/tests/qa-test-pr.bats` locally on main immediately after any script-touching PR merges.**

## BATS test coverage — known gaps fixed in PR #675

- `scripts/tests/qa-test-pr.bats` — `domain:iso` tier-3 routing was untested. Now covered.
- `scripts/tests/build-iso.bats` — `--binary` arg forms and `arm64 + stable/beta/alpha/edge` combos were untested. Now covered.

## Blocked quality issues (as of 2026-06-01)

| Issue | Blocked on |
|---|---|
| #654 — `cmd/nvidia-check` unit tests | PR #652 (`nvidia-check` tool) must merge first |
| #655 — headless/wizard FCOS OS-branching tests | PR #653 (FCOS headless `os` field) must merge first |

When those PRs land, pick up the issues immediately — both are small, self-contained test additions (~60 lines each).
