# FSC Patch

FSC Patch inspects, patches, or restores the FSC verifier on a supported BMW
MGU head unit. Version `1.0.0` recognizes one inspected BAC4x verifier build;
unknown or modified builds are refused before a live change.

> [!WARNING]
> Patching changes a system binary and persistent FSC data. An interruption,
> incompatible firmware, or incorrect recovery can make navigation or other
> head-unit functions unavailable. Review the source and keep a recoverable
> backup before making changes.

## Firmware compatibility

Before any patch, the service reads the running firmware version (from BMW
version files or the verifier binary's SHA-256) and checks it against the
compatibility registry. Only registered firmware is patched; every other build
is refused with a clear message.

| Firmware | Status | Notes |
|----------|--------|-------|
| BAC4x VSTAR i386 Eth Application (SHA-256 `b26d1830...`) | Supported | v1.0.0 inspected build |
| P18_23w44.2-1-3 mgu-high | Not yet registered | See below |

The `P18_23w44.2-1-3` firmware reported in [ISSUE #2](../ISSUE.md) is **not**
in the default compatibility registry. To patch it:

1. Extract `/usr/bin/BAC4x_VSTAR_i386_Eth_Application.out` from the head unit.
2. Compute its SHA-256: `sha256sum BAC4x_VSTAR_i386_Eth_Application.out`
3. Add a `FirmwareProfile` entry in `native/mgu-fscpatch/data_embedded.go`
   with the hash, size, stock pattern, and modulus offsets.
4. Rebuild and reinstall the `mgu-fscpatch` service.

Until a profile is registered, the Patch button is disabled and the service
refuses to modify any files — this is by design, to prevent the transitional
verifier state that caused boot loops.

For the supported build, FSC Patch:

- keeps the factory verification key and factory FSC path intact;
- adds a fallback verification path for the intentionally public local key;
- preserves existing FSC records while merging the local support certificate;
- creates and verifies backups before changing the verifier or support block;
- supports restoring the original files; and
- stages persistence changes for the next head-unit restart.

The app uses the Store-managed
[`mgu-fscpatch` service](../../native/mgu-fscpatch/README.md). Its local
root-only control socket performs patch, restore, and restart operations after
confirmation in the UI.

## Recovery from boot loop

If the head unit enters a boot loop after patching (as reported in
[ISSUE #2](../ISSUE.md)), use this recovery path:

1. **Catch the SSH window** during a boot cycle (port 22 may be open for only
   a few seconds).
2. **Disable the FSC patch service:** `systemctl disable --now mgu-fscpatch.service`
3. **Perform a factory reset of the HU_MGU via ISTA** (including removal of
   personal settings / CAFD reset). This breaks the boot loop.
4. **Re-install MGU Store from scratch.** Apps are pulled back automatically
   from the sources.
5. The FSC Patch service will re-apply cleanly on the clean baseline.

The boot loop was caused by a transitional verifier state between patched and
un-patched code paths after a crash, not by the patch logic itself. The new
pre-patch firmware validation and boot-time guard in v1.0.1+ prevent this by
refusing unknown firmware and retaining pending jobs for inspection instead of
committing them across a firmware mismatch.

The separate [FSC generator](../../scripts/fsc_generator/README.md) creates
application and ID7 Live-map FSC files for the local key accepted by the
patch. Its signing key is deliberately public interoperability material, not
an access credential.

## Build and test

Requirements:

- Node.js 24 or later
- Python 3.12 or later
- Go 1.23 or later

From the repository root:

```sh
npm test
npm run test:fsc
npm run test:generator
npm run build:fscpatch
npm run build:apps
```

The app package and native service are committed with source fingerprints and
checksums. Rebuild the affected artifact after changing the app, shared kit,
service, or native payload.
