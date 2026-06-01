# VM Testing Patterns

Load when: writing or running vm-e2e tests, debugging QEMU installs, working on ISO builds.

## Quick Reference — Justfile Recipes

```bash
just vm              # interactive TUI: build → deploy → install in QEMU
just vm-e2e          # automated 4-pass: DHCP · static · sysext · NVIDIA
just iso [channel]   # build UEFI installer ISO
just iso-smoke <iso> <ovmf> <timeout>   # headless serial-log smoke test
just boot-iso        # boot ISO on serial console (Ctrl-a x to quit)
just e2e             # build ISO + launch in Ghostty window (local display required)
just headless-test   # no VM needed — exercises config generation only (CI gate)
just stop            # kill running VM
just clean           # kill VM + remove all artifacts
```

## How `just vm` Works

1. Builds binary (`linux/amd64`, CGO_ENABLED=0)
2. Creates qcow2 overlay on cached Flatcar base image (instant, no 3 GB copy)
3. Creates 20 GB target disk
4. Boots QEMU daemonized with port-forward (2222→22)
5. Waits for SSH (~6s with KVM)
6. SCPs binary to `/tmp/knuckle`
7. SSHes into the VM — run knuckle interactively (writes to `/dev/vdb`)
8. After install: kills installer VM, boots installed target disk automatically

⛔ **Never embed the binary in Ignition via base64** — 19 MB → 26 MB JSON, breaks `fw_cfg`.

## How `just vm-e2e` Works

4 automated passes, each with a fresh qcow2 overlay:

| Pass | Key assertions |
|---|---|
| DHCP | hostname, update strategy, `core` user groups |
| Static | `/etc/systemd/network/10-static.network` content (address, gateway, interface) |
| Sysext | `docker.raw` present + size, `systemd-sysext` active, `docker version` |
| NVIDIA | `/etc/flatcar/enabled-sysext.conf` contains `nvidia-drivers-*` |

Each pass has a 15–25 min timeout. `just vm-e2e` tees output to `/tmp/vm-e2e-run.log`.

## ISO Architecture

- **Kernel:** `flatcar_production_pxe.vmlinuz` from Flatcar CDN
- **Initrd:** `flatcar_production_pxe_image.cpio.gz` + knuckle overlay cpio (appended)
- **Boot:** GRUB standalone EFI (`grub-mkstandalone` with fat/part_gpt/search/linux modules)
- **Assembly:** xorriso with El Torito EFI boot image
- **Overlay:** `/opt/knuckle` binary + `knuckle-installer.service` systemd unit
- **UEFI only** (no BIOS/legacy boot)
- **Build deps:** `x86_64-elf-grub-mkstandalone`, `xorriso`, `mtools` (mformat/mcopy), `cpio`

## ISO Boot Smoke Test

`just vm-e2e` does **not** test ISO boot — it installs via headless mode directly. ISO boot is tested by the `iso-boot-smoke` GHA job in `ci.yml`.

Serial log invariants (checked by `iso-smoke.sh`):
- `systemd.gpt_auto=0` must appear on **both** BLS entries (primary + serial)
- `initrd-root-device.target`, `initrd-usr-fs.target`, `getty.target` must appear
- `x2dauto` / `xd2root.device` / `dracut.*skip` must NOT appear

**Why `systemd.gpt_auto=0` matters:** Without it, bare metal GPT disks trigger `systemd-gpt-auto-generator` → dracut `xd2root` hook is skipped → boot failure. Root cause of v0.6.2 bare metal issue (fixed in v0.7.0).

## Agent Verification Limitations

| Can verify | Cannot verify |
|---|---|
| Binary builds (`go build`) | Forms render correctly |
| Unit tests pass (`go test`) | User can navigate steps |
| Process launches (`pgrep`) | Fields accept input |
| VM boots (SSH works) | Install progress animates |
| Installed system config | TUI doesn't crash mid-flow |
| Headless mode output | Interactive experience |

**Protocol:** Launch a terminal for the user; say "launched — awaiting your feedback." Never claim "verified" for TUI behavior.

## Gotchas

| Problem | Fix |
|---|---|
| `just e2e` / `just boot-iso` fail on headless host | These recipes use `-display gtk` — local display required |
| VM port 2222 in use | `just stop` |
| SSH after ISO boot fails | Live ISO has no SSH key — use `just vm` for SSH testing |
| ISO doesn't boot (EFI shell) | Need OVMF firmware: `-drive if=pflash,format=raw,readonly=on,file=$OVMF` |
| GRUB "file not found" | Needs `search --file /vmlinuz --set=root` in `grub.cfg` |
| Base image missing | First `just vm` downloads ~480 MB Flatcar image (cached in `.vm/`) |

## Never Boot the Base Image Directly

When reproducing vm-e2e issues, do **not** boot `.vm/flatcar_base_<arch>.img` as a writable disk. Doing so mutates first-boot state and causes later runs to skip Ignition key injection, which looks like random SSH auth failures.

Always use a fresh qcow2 overlay:
```bash
qemu-img create -f qcow2 -b "$(pwd)/.vm/flatcar_base_amd64.img" -F qcow2 .vm/boot.qcow2
```

If a base image was accidentally booted directly, delete it — `just _ensure-base` will redownload a clean copy.

## vm-e2e: Disable Swap for Sysext/NVIDIA Passes

Headless config defaults swap to enabled when `swap` is omitted. For sysext and NVIDIA passes, unrelated boot ordering noise from `systemd-sysext` cycle messages can obscure assertions. Set explicitly:

```json
"swap": {"enabled": false}
```

## vm-e2e GHA: JSON Field Names Are Exact

Go's `encoding/json` **silently ignores unknown fields**. Always cross-reference `docs/HEADLESS-CONFIG.md` for canonical field names.

```json
// ✅ Correct NVIDIA config
{"nvidia_driver_version": "570-open", "swap": {"enabled": false}, ...}

// ❌ Wrong — silently ignored, NVIDIA not configured
{"nvidia": {"enabled": true, "driver_type": "open"}, ...}
```

## vm-e2e GHA: Assertion Paths Must Match Butane Output

Check `internal/ignition/ignition.go` (`butaneTemplate` const) for the exact file paths written before writing any `$E2E_SSH "test -f ..."` assertion.

Examples:
- NVIDIA writes to `/etc/flatcar/enabled-sysext.conf` — **not** `/etc/sysupdate.d/`
- Static network config writes to `/etc/systemd/network/10-static.network`

## vm-e2e Port-Forward Constraint

`-net user,hostfwd=tcp::2222-:22` binds `127.0.0.1:2222` on the machine running QEMU. If QEMU is on a remote host:

```
✅ SSH to remote host → then ssh -p 2222 core@127.0.0.1  (from remote)
❌ ssh -p 2222 remote-host                               (reaches host sshd, not VM)
```
