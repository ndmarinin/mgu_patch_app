# FSC patch service

Version 1.0.0 supports the inspected BAC4x build with stock SHA-256
`b26d18308c0f2ca1edd5aff8c373aa3aa659751e940cb6eed8509873670c1231`.
Unknown builds are refused before live changes. A matching stock binary backup
is required to repair an incomplete patch.

## Firmware compatibility registry

The service maintains a firmware compatibility registry in
`native/mgu-fscpatch/data_embedded.go`. Before any patch, it:

1. Detects the running firmware from BMW version files (e.g. `/etc/version`)
   or falls back to the verifier binary's SHA-256.
2. Looks up the firmware in the registry.
3. Refuses to patch if the firmware is unknown or the binary hash/size does
   not match the registered profile.

To add support for a new firmware build:

1. Extract `/usr/bin/BAC4x_VSTAR_i386_Eth_Application.out` from the head unit.
2. Compute its SHA-256 and size.
3. Find the stock pattern (anti-drift prefix at the patch site) and the
   local public modulus offsets.
4. Add a `FirmwareProfile` entry in `data_embedded.go`'s `firmwareRegistry`.
5. Map a human-readable version string (e.g. `"P18_23w44.2-1-3"`) to the
   profile key in `firmwareIDByVersion`.
6. Rebuild and reinstall the service.

Until a profile is registered, the Patch button is disabled and the service
refuses all modifications — preventing the transitional verifier state that
caused boot loops on unregistered firmware.

## Snapshot manifests

Every successful patch writes a snapshot manifest to
`/var/opt/mgu-fscpatch/fscpatch-snapshot-<timestamp>.json` recording:

- Firmware ID and name
- Binary SHA-256 and size
- Block SHA-256 and presence
- Whether the binary was already patched
- Creation timestamp

Restore operations verify that the current system state matches a snapshot
before restoring backups, preventing cross-firmware restores.

## Boot-time guard

The `autosaronlinux.service` drop-in runs `mgu-fscpatch --commit-block` as
`ExecStartPre` after persistence mounts and before the verifier loads NVM.
If a pending support-block job's binary hash does not match the installed
binary, the job is retained for inspection and no change is committed. This
prevents a stale patch job from being applied across a firmware change or
interrupted patch.

The factory modulus stays intact. A position-independent x86-64 wrapper calls
the original CAL signature verifier with the factory key first. Only an invalid
signature (`0xf9`) for the FSC key descriptor permits a second call with the
local public key. The original caller retains parsing, VIN, application,
upgrade and validity policy. Other key descriptors do not use the fallback.
The entire patched image must match the reconstructed known-stock profile,
including the payload, program header, detour, padding and trailer.

Support configuration is merged into the 30-record NVM block. Existing FSC
records are preserved byte-for-byte; missing originals can be recovered from
the retained support-block backup. Current certificates take priority over
backups of the same software identifier. A malformed block or insufficient
space aborts the operation. Deleted certificates cannot be recreated without
an original record or backup.

The service writes a one-time, root-owned pending job tied to the installed
binary's SHA-256. An autosaronlinux.service ExecStartPre drop-in commits it
after persistence mounts and before NVM is loaded. It merges certificates
flushed by the old running verifier during shutdown. Restore stages the
original block instead. Completed jobs are removed, firmware mismatches leave
the job for inspection, and a hook failure never prevents AUTOSAR startup.
`blockPending` and `restartRequired` expose the staged state to the app.

The app's Restart head unit action needs a second press within eight seconds.
POST /restart requests `/usr/bin/nsm_control --requestRestart 7 --normal`, using
the same unit restart mechanism as the Store. NSM refusal and timeout are
reported. Restart shares the operation mutex and root/method/body checks.

### HTTP API

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/status` | GET | Full status including binary, block, backups, and firmware compatibility |
| `/patch` | POST | Apply patch (requires root, double-confirmed in UI) |
| `/restore` | POST | Restore from backup (requires root) |
| `/firmware` | GET | Detected firmware and compatibility status |
| `/check` | GET | Dry-run validation: reports whether patch would be accepted, without writing |

Validation: the native suite runs under Linux 4.14.328 in QEMU with a temporary
ext4 disk, read-only tmpfs mounts and executing ELF fixtures. A production-code
round trip patches the captured known build and restores it byte-for-byte.
An offline harness executes the patched image's original CAL routines with
four captured factory FSCs and rejects altered copies. A disposable RSA test
key exercises the same fallback instructions with its public modulus
substituted only in the test image. No private key or vehicle file is packaged
in the app or service. The separate offline generator intentionally publishes
the matching local signing key under `scripts/fsc_generator/`.
Physical head-unit restart/persistence validation remains pending.

The custom wrapper source is in payload/dual.c and payload/dual.ld. It was
built with GCC 11 (`-Os -fPIC -fno-stack-protector
-fno-asynchronous-unwind-tables -ffreestanding -fno-builtin`), linked with GNU
ld using dual.ld, and extracted using objcopy -O binary. The embedded payload
SHA-256 is `e3b729890e619d1c5e24d069c301facbd4df35699cee48d3f040075ceb0fbe6d`.

The local key's little-endian modulus SHA-256 is
`d4f1ebaece4e4dbc1536a28eaa15b6b9f80f50213bd485ba01495fb2c6c469e0`.
The installed verifier accepts OEM and local signatures; factory FSCs keep the
OEM verification path.
