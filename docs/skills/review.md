# PR Review Workflow

Load when: reviewing any knuckle PR, deciding tier classification, running vm-e2e.

## Pre-Flight (once per session)

```bash
# Check for first-time contributor PRs stuck at action_required
gh api repos/projectbluefin/knuckle/actions/runs?status=action_required \
  --jq '.workflow_runs[] | "\(.id) \(.name) \(.head_branch)"'
# If any: gh api repos/projectbluefin/knuckle/actions/runs/<ID>/approve --method POST
# Must approve CI and Security runs separately — they have different run IDs
```

## Tier Classification

Tier is set by the **highest-tier domain label** present. `kind/test` alone = Tier 0.
**Never use PR title to determine tier** — use labels only.

| Labels | Tier | What runs |
|---|---|---|
| `domain:ci`, `kind/test`, docs | 0 | `just ci` |
| `domain:probe`, `domain:tui` | 1 | Tier 0 + VM tool check + dry-run |
| `domain:security` | 1+sec | Tier 1 + bad-input rejection tests |
| `domain:install`, `domain:headless`, `domain:ignition`, swap, tailscale, sysext | **3** | Tier 1 + full install + boot installed system + domain assertions (`just vm-e2e` or GHA `vm-e2e.yml`) |
| `domain:iso` | 3 | Tier 3 + hardware-repro |

## Complexity Gate (skip vm-e2e if ANY)

| Signal | Threshold |
|---|---|
| `size:XL` or `size:XXL` label | present |
| Domain labels | >4 distinct `domain:*` |
| Workflow files | any `.github/workflows/*.yml` changed |
| Architecture boundary | `cmd/knuckle` + `internal/runner` + `internal/ignition` together |

## Code Review Checklist

```
□ gofmt clean: double space before // is the most common failure
□ No exec.Command outside internal/runner
□ Disk identity via /dev/disk/by-id (not /dev/sdX)
□ Ignition tempfile: os.CreateTemp + chmod 0600 + defer os.Remove
□ No secrets in slog output
□ Test assertions check err.Error() content, not just err != nil
□ Permission tests skip with t.Skip if os.Getuid() == 0
□ Every LGTM backed by a file:line reference from the diff
```

## Domain-Specific Review Patterns

| Domain | Key check |
|---|---|
| `install` | `wipefs → flatcar-install → sfdisk` order; DryRunner no-ops all three |
| `ignition` | `{{- end}}` balanced; `yamlEscape` on every user string |
| `headless` | `Validate()` called before `ToInstallConfig()`; SSH keys validated |
| `tui` | No business logic in view model; `wizard.Apply*` for mutations |
| `validate` | Table-driven tests; error messages include the bad value |
| `wizard` | Conditional steps check selector in Next/Previous/GoToStep |
| `bakery` | SHA512 + GPG both checked; no per-call `http.Client` |
| `ci/release` | `persist-credentials: false` on all checkout steps |

## VM E2E — Run Options

**Option A — GitHub Actions (preferred for Tier 3 PRs):**
```bash
gh workflow run vm-e2e.yml --repo projectbluefin/knuckle --ref <branch>
gh run list --repo projectbluefin/knuckle --workflow vm-e2e.yml --limit 3
```

**Option B — Local:**
```bash
cd ~/src/knuckle
git checkout <pr-branch>
just vm-e2e   # runs 4 passes: DHCP → static → sysext → NVIDIA
```

Requires `/dev/kvm` + QEMU installed. Flatcar base (~480 MB) is cached after first run.

## Decision Protocol — ONE COMMENT RULE

Post the strike report once as a PR comment. Never post a follow-up for a new observation — **edit the existing comment**.

```bash
# GO — post report, approve (no body), queue
gh pr comment <N> --repo projectbluefin/knuckle --body-file /tmp/qa-stdout-<N>.txt
gh pr review <N> --repo projectbluefin/knuckle --approve
gh pr merge --auto <N> --repo projectbluefin/knuckle

# NOGO — post report, request changes (gh requires a non-empty body)
gh pr comment <N> --repo projectbluefin/knuckle --body-file /tmp/qa-stdout-<N>.txt
gh pr review <N> --repo projectbluefin/knuckle --request-changes \
  --body "See strike report comment for requested changes."
```

⛔ **Always `gh pr merge --auto`**. Direct merge bypasses CI on the combined branch.
⛔ **You cannot self-approve** — `gh pr review --approve` fails with "Can not approve your own pull request" when you authored the PR. Leave the strike report and note manual approval required.

## Workflow File PRs

`.github/workflows/*.yml` PRs **cannot be auto-merged** — require `workflow` OAuth scope.
These are merged manually via GitHub UI. Approve + leave; do not attempt `gh pr merge`.

## Worktree Hygiene

Worktrees from prior sessions accumulate in `/tmp/knuckle-pr-*`. Run cleanup at the end of every batch session:

```bash
cd ~/src/knuckle
for wt in $(git worktree list --porcelain | grep worktree | awk '{print $2}' | grep /tmp/knuckle-pr-); do
  git worktree remove "$wt" --force 2>/dev/null && echo "removed $wt"
done
git worktree list  # verify clean
```

## PR Scope Gate (agent-authored branches)

After every agent-authored PR, verify scope immediately:
```bash
gh pr view <N> --repo projectbluefin/knuckle --json commits,files
gh pr diff <N> --repo projectbluefin/knuckle --name-only
```

If the commit or file list includes unrelated paths, **replace the PR**: clean branch from `upstream/main`, cherry-pick only the intended commit, open replacement, close the original with a pointer.

## Common Failures

| Failure | Cause | Fix |
|---|---|---|
| `go: updating go.mod: existing contents have changed` | Parallel QA runs race on go.mod | Run QA scripts **sequentially**, never in parallel |
| `open /dev/tty: no such device or address` (cmd/knuckle) | No PTY in non-interactive shell | Pre-existing; tracked as issue #512. Rely on GitHub CI for Tier 0 |
| `INSTALL_FAILED` | `flatcar-install` non-zero | Read install log in report |
| `INSTALLED_BOOT_TIMEOUT` | Ignition failed at first boot | Check Ignition errors in knuckle-install.log |
| `git index.lock` | Multiple scripts checkout concurrently | One git worktree per PR — run sequentially |
| `git worktree add` fails | Stale worktree from prior run | `git worktree remove /tmp/knuckle-qa-wt-<N> --force` before rerun |
| `git fetch ... pr<N>-qa` exits 128 | Stale local ref | `git update-ref -d refs/heads/pr<N>-qa` then rerun |
| PR stuck `BLOCKED`, no CI | First-time contributor — GitHub holds runs | Approve via `gh api repos/.../actions/runs/<ID>/approve --method POST` |
| Can't self-approve | Agent authored the PR | Leave strike report; flag manual approval needed |

## File Overlap = Sequential Queue

Check before queueing any two PRs:
```bash
gh pr diff <N1> --name-only
gh pr diff <N2> --name-only
```
If they touch the same file, queue sequentially — the merge queue will conflict otherwise.

## Remote Setup (fork pattern)

The upstream repo is `projectbluefin/knuckle`. Branch workflow:
```bash
git checkout -b fix/slug upstream/main   # always base on upstream/main
git push origin fix/slug                 # push to fork
```
Compare URL: `https://github.com/projectbluefin/knuckle/compare/main...<fork>:knuckle:<branch>`
