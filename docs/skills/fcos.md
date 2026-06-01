# FCOS Implementation Guide

Load when: working on any FCOS epic issue (#500, #637–#645).

## Dependency order — strict, do not skip

Issues must land in this order. Each tier can be started only after all issues in the previous tier are merged.

```
Tier 1 (parallel, no deps):
  #637 — feat(validate): FCOS stream validation, gate arm64/lts on Flatcar
  #638 — feat(install): OS-dispatching installer and bakery client wiring
       ↓
Tier 2 (after #638, can be parallel):
  #639 — feat(ignition): FCOS Butane variant + zincati update config
  #640 — feat(install): FCOSInstaller via coreos-installer
  #641 — feat(bakery): FCOS sysext catalog client (fedora-sysexts/community)
       ↓
Tier 3 (after their specific deps):
  #642 — feat(headless): FCOS os field + stream validation   (needs #637+#638+#639+#640)
  #644 — feat(iso): FCOS live ISO via coreos-installer       (needs #640)
       ↓
Tier 4 (after all impl):
  #643 — feat(tui): OS selection + FCOS-conditional steps    (needs #642+#644)
       ↓
Last:
  #645 — test: FCOS test coverage across all packages        (needs all of the above)
```

## Why #638 is the critical first step

`cmd/knuckle/main.go` constructs the installer and bakery client **before** the OS is known. #638 adds the OS-dispatching factory so downstream issues (#639, #640, #641) have the interface to implement against. Starting any of those without #638 means rewriting the wiring twice.

**Pattern for any feature that branches on user input from StepWelcome:** Use a `DispatchingInstaller` that holds both impls and delegates at `Install()` call time based on `cfg.OS`:

```go
installer = &install.DispatchingInstaller{
    Flatcar: install.NewFlatcarInstaller(cmdRunner, logger),
    FCOS:    install.NewFCOSInstaller(cmdRunner, logger),
}
```

Same pattern applies to the bakery client — the wizard must call `FetchCatalog*` based on `cfg.OS` at `StepSysext`, not at startup.

## FCOS ISO: use `coreos-installer iso customize`

```bash
# ✅ Correct: embed knuckle into FCOS live ISO
coreos-installer iso customize \
  --dest-ignition installer.ign \
  --output out.iso fcos-live.iso

# ❌ Wrong: `pxe customize` is for PXE images, not ISO
coreos-installer pxe customize ...
```

FCOS live image runs `getty@tty1.service` with autologin for `core`. The knuckle service unit must add `Conflicts=getty@tty1.service` and `Before=getty@tty1.service` or the TUI will not render.

## coreos-installer vs flatcar-install

| | `flatcar-install` | `coreos-installer` |
|---|---|---|
| Sub-command | (direct flags) | `bare-metal install` |
| Image source | `-d <disk>` | `--dest-device <disk>` |
| Ignition | `-i <file>` | `--ignition-file <file>` |
| Architecture | `-A <arch>` | `--architecture <arch>` |
| Channel/stream | `-C <channel>` | (stream baked into image URL) |

Never mix flags between the two tools. #640 implements `FCOSInstaller` — look there for the canonical invocation.

## FCOS stream names (validate.FCOSStream)

Valid FCOS streams: `stable`, `testing`, `next`. Note `lts` and `edge` are Flatcar-only — `validate.FCOSStream()` rejects them. `arm64 + lts` is only blocked for Flatcar, not FCOS.

## Test coverage issues

#645 is explicitly listed as "do last" in its own issue body. Do not open #645 PRs until all of #637–#644 are merged. Each implementation issue should include its own unit tests inline — #645 adds integration-level coverage only.
