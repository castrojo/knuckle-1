# Release Workflow

Load when: cutting any knuckle release, running the pre-release E2E gate, auditing release assets.

## Release Asset Inventory — Expected 16 Total

Each release ships **8 × amd64 + 8 × arm64**:

| Asset | ×amd64 | ×arm64 |
|---|---|---|
| `knuckle-linux-<arch>` | ✅ | ✅ |
| `knuckle-linux-<arch>.sha256` | ✅ | ✅ |
| `knuckle-linux-<arch>.spdx.json` | ✅ | ✅ |
| `knuckle-linux-<arch>.spdx.json.bundle` | ✅ | ✅ |
| `knuckle-installer-stable-<arch>.iso` | ✅ | ✅ |
| `knuckle-installer-stable-<arch>.iso.sha256` | ✅ | ✅ |
| `knuckle-installer-stable-<arch>.iso.bundle` | ✅ | ✅ |
| `knuckle-linux-<arch>.bundle` | ✅ | ✅ |

```bash
# Audit after release publishes — must be exactly 16 lines
gh release view vX.Y.Z --repo projectbluefin/knuckle --json assets \
  --jq '.assets | map(.name) | sort | .[]'
```

## Pre-Release E2E Gate

### Step 1 — CI Gate
```bash
git checkout main && git pull upstream main
just ci
```
All checks must pass. If `cmd/knuckle` TTY tests fail with "open /dev/tty", that is pre-existing infra issue #512 — does NOT block release. All other failures block.

### Step 2 — Release Preflight
```bash
just release-preflight
```
Runs: `just ci` + sysext catalog coverage check + NVIDIA driver series check. Must exit 0.

### Step 3 — VM E2E (4 passes)
```bash
just vm-e2e   # runs DHCP · static · sysext · NVIDIA
```
All 4 passes must exit 0. Any `FAIL` blocks the release.

### Step 4 — amd64 ISO Smoke
```bash
just build
just iso   # builds output/knuckle-installer-stable-amd64.iso

OVMF=$(ls /usr/share/OVMF/OVMF_CODE*.fd /usr/share/edk2/ovmf/OVMF_CODE*.fd 2>/dev/null | head -1)
just iso-smoke output/knuckle-installer-stable-amd64.iso "$OVMF" 120
```
Pass criteria: `initrd-root-device.target`, `initrd-usr-fs.target`, `getty.target` in serial log; zero `xd2root`/`x2dauto`/`dracut.*skip` errors; `systemd.gpt_auto=0` on both BLS entries.

### Step 5 — arm64 ISO Smoke (TCG, manual)
No native KVM on most hosts for arm64 — uses TCG (software emulation). Too slow for CI; run manually before tagging. Evidence (quoted log excerpt) must be recorded in the release epic before tagging.

```bash
KNUCKLE_ARCH=arm64 just build
KNUCKLE_ARCH=arm64 just iso

AAVMF=$(ls /usr/share/AAVMF/AAVMF_CODE.fd /usr/share/qemu-efi-aarch64/QEMU_EFI.fd 2>/dev/null | head -1)
timeout 300 qemu-system-aarch64 \
  -M virt -cpu cortex-a57 -m 2048 \
  -drive if=pflash,format=raw,readonly=on,file="$AAVMF" \
  -cdrom output/knuckle-installer-stable-arm64.iso \
  -drive if=virtio,file=/tmp/arm64-smoke-target.img,format=raw \
  -nographic 2>&1 | tee /tmp/arm64-iso-smoke.log || true
```

Pass criteria: same as amd64. 5 min timeout (TCG is ~5× real-time).

## Tag & Publish

```bash
git status       # must be clean
git log --oneline -5
git tag vX.Y.Z
git push upstream vX.Y.Z   # triggers release.yml
gh run watch --repo projectbluefin/knuckle
```

The release.yml pipeline runs parallel arch builds:
```
create-release ──┬─→ release-amd64 ──┐
                 └─→ release-arm64 ──┴→ publish
```

⛔ **Do NOT push the same tag twice.** The `publish` job will fail on asset upload to an immutable release.

## Post-Release Verification

```bash
# 1. Asset count (must be 16)
gh release view vX.Y.Z --repo projectbluefin/knuckle --json assets \
  --jq '.assets | length'

# 2. Both arch ISOs present
gh release view vX.Y.Z --repo projectbluefin/knuckle --json assets \
  --jq '.assets | map(.name) | map(select(test("arm64|amd64"))) | sort | .[]'

# 3. Cosign bundles (must be 4: 2 per arch — binary + iso)
gh release view vX.Y.Z --repo projectbluefin/knuckle --json assets \
  --jq '.assets | map(select(.name | endswith(".bundle"))) | length'
```

## Known Failure Modes

| Symptom | Cause | Fix |
|---|---|---|
| arm64 assets missing | `release-arm64 needs release-amd64` race (pre-PR-#606) | Ensure parallel-build release.yml is on main before tagging |
| `Cannot upload asset … to an immutable release` | Tag pushed twice | Delete release + tag, fix, re-tag |
| `arm64 build failed` cross-compile | `CGO_ENABLED` not 0 on arm64 runner | Check `CGO_ENABLED=0` in all arm64 build steps |
| `iso-smoke` hangs | Missing `systemd.gpt_auto=0` on BLS entries | Check `scripts/build-iso.sh` — both `knuckle.conf` and `knuckle-serial.conf` must have it |
| OSSF Scorecard skipped on release | `scorecard` job only runs on push to `main`, not tags | Expected — not a blocker |
| Workflow run blocked (first-time contributor) | GitHub holds workflow runs | Approve via `gh api repos/projectbluefin/knuckle/actions/runs/<ID>/approve --method POST` |

## Lessons Learned

### v0.7.0 — arm64 artifacts missing
`release-arm64` had `needs: release-amd64`. On tag retry, first run's `publish` made the release immutable before arm64 could upload. Fixed in PR #606 (parallel `create-release` gate). Never tag before the parallel-arch release.yml is on main.

### v0.6.2 — ISO boot failure on bare metal
`systemd.gpt_auto=0` missing from BLS entries. Bare metal GPT disks triggered `systemd-gpt-auto-generator` → dracut `xd2root` hook skipped. Fixed in v0.7.0. `just iso-smoke` now catches this in CI.

### `just vm-e2e` does NOT test ISO boot
`vm-e2e` deploys the knuckle binary via SSH to a running Flatcar VM. It does **not** boot from the installer ISO. Use `just iso-smoke` or `just boot-iso` for ISO boot validation. Both are required for a complete release gate.
