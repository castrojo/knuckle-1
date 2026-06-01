---
name: knuckle-skills
description: Entry point for projectbluefin/knuckle skill tree. Load this first for any knuckle task; load sub-skills on demand.
---

# knuckle skill index

## Hard rules (enforced by `just ci`)

1. **All external commands via `internal/runner`** — no `exec.Command` outside that package.
2. **Never run `flatcar-install` on the host** — only inside QEMU via `just vm` / `just vm-e2e`.
3. **Disk identity via `/dev/disk/by-id`** — never trust `/dev/sdX` enumeration order.
4. **Never log to stdout** — Bubble Tea owns it. Use `log/slog` with a file handler.
5. **`just ci` is the gate** — tidy + fmt + vet + lint + vuln + race + cover-check + headless-test + shell-lint + build. Never `--no-verify`.

## Load on demand

| Task | Load |
|---|---|
| Coverage gates, test pyramid, CI pipeline, ISO build internals | `docs/CI-AND-TESTING.md` |
| PR test matrix, tier classification, domain assertions | `docs/PR-TEST-MATRIX.md` |
| Ghost lab setup, KubeVirt, QA workflow | `docs/GHOST-LAB.md` |
| Release checklist, VM verification | `docs/RELEASE.md` |
| Headless config schema, field reference, validation rules | `docs/HEADLESS-CONFIG.md` |
| Sysext catalog, Bakery tiers, extension behavior | `docs/SYSEXTS.md` |
| Troubleshooting runbook, first-boot diagnostics | `docs/TROUBLESHOOTING.md` |
| Butane-as-library rationale | `docs/BUTANE-DEPENDENCY.md` |
| Coverage gaps, quality agent patterns, stale-issue gotchas | `docs/skills/quality.md` |
| FCOS implementation order and dependency chain | `docs/skills/fcos.md` |
| PR review workflow, tier classification, vm-e2e decision | `docs/skills/review.md` |
| VM testing patterns, ISO architecture, QEMU gotchas | `docs/skills/testing.md` |
| Release E2E gate, asset audit, known failure modes | `docs/skills/release.md` |
