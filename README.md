# Sony Xperia XZ2 Premium SOV38: unlock and OrionOS installation journal

## Scope and result

This documents one successful SOV38 KDDI session, reported on 2026-10-21. It is an evidence-based journal plus a guarded runbook, **not a universal unlock recipe**. The final result was persistent bootloader unlock and the OrionOS setup/Hello screen after Google apps installation and a user-confirmed data format.

The starting phone ran KDDI Android 10 `52.1.C.0.268`, bootloader Q207, active slot A, locked, with no unlock eligibility. After the exploit and a manual reset, standard `fastboot getvar unlocked` returned `yes`. No Sony unlock code was used. The service-menu eligibility field was not checked afterward; do not claim it changed to Yes.

Hardware functions, DRM, NFC, Netflix, attestation, and long-term stability were not validated. Do not describe the result as fully tested or DRM-preserving merely because upstream describes that capability. OrionOS is identified by its published project/package; this journal makes no claim that it is a renamed Matrixx build.

### Evidence boundaries

- Session outcomes, hashes, build revisions, and byte totals below come from the recorded session summary supplied for this document.
- Local inspection checked `backup_sov38.py`, `sync_b_to_a.py`, and upstream `xperable/README.rst` for implementation and command semantics. Neither Python script was executed while writing this guide.
- External HDD artifacts and original session logs were not re-read or independently rehashed for this documentation pass. Treat recorded hashes as artifact identifiers, not a fresh verification of files on your disk.
- No phone operations were performed to create this document. Only this README was created; no script, executable, or commit was changed.

## Read before attempting anything

**Unlocking, bootloader flashing, partition copying, and formatting can permanently destroy data or leave the phone unbootable. Back up personal data before starting. Assume encrypted userdata can become inaccessible even if an unlock operation does not explicitly erase it.**

- Use only a device you own or are authorized to modify.
- Maintain adequate battery charge, stable supply voltage, a reliable cable/USB port, and uninterrupted host power. Do not flash an unstable or intermittently disconnecting device.
- Match the exact model, firmware lineage, boot manifest, and intended slot. Do not arbitrarily cross-flash international XZ2/XZ3 firmware onto SOV38.
- **Never relock as part of this procedure, especially with custom or mixed-slot firmware.** Upstream relock examples are not instructions for this installation.
- EDL recovery availability and a working emergency restore path are unknown. A backup does not establish that a hard-bricked phone can be restored.
- Stop on unexpected identity, bootloader revision, geometry, slot state, hash, transport error, or verification failure. Do not substitute repeated writes for diagnosis.
- Recovery access and a blue fastboot LED are not proof of a bootable Android system. A running root ADB recovery can have a blank screen.
- Keep serial numbers, IMEI, TA contents, raw partition images, device keys, and unredacted logs private. The existing local scripts are device-bound and must not be published as generic tools without sanitization.

## Artifact ledger

| Component | Recorded identity / role |
| --- | --- |
| Original installed stock | SOV38 KDDI Android 10 `52.1.C.0.268`, Q207, slot A |
| Matching stock package used for bootloader preparation | `SOV38_KDDI_JP_52.0.C.5.295-R13B`; boot manifest `aurora`, commercial `B6B5`, P118 |
| Stock package local location | `O:\SonyStuff\Stock_Firmware\SOV38_KDDI_JP_52.0.C.5.295-R13B` |
| newflasher | Version 58, commit `2da5384c71df0fdf924bf3a5ad275f6bef59ce44` |
| xperable | Upstream commit `0bf7060a966f91ed6251386bb6281648e416a03a`; unpatched P118 implementation, built as a Windows executable |
| Exploit support blob | `LinuxLoader-p118.pe`; SHA-256 below |
| Windows USB driver | Native WinUSB installed using official Zadig 2.9 |
| Temporary TWRP | j4nn `twrp-3.7.1_12-0_LOCAL-20251206-01-aurora.img` |
| ROM | OrionOS aurora 15.1, published directory `20250303`; local ZIP under `CUSTOMROM` |
| Google apps | Official MindTheGapps Android 15 arm64, build `20260915_032013` |

Recorded SHA-256 values:

```text
LinuxLoader-p118.pe
14fb6ddeef4086f76145ae25a1987f7f9af5243f3edf47efadcd22702220ed3a

OrionOS ROM ZIP
5a5f2183fe1bee06874c4c2f484dfffeb062c60d24764a9e50282b78863449f3

Orion external boot image
3633a28be82f3c8dc9b859646ab1a82299d25e176495b6804d69715535f0e2de

MindTheGapps 15 arm64 20260915_032013
e979601ef70b03d9214ea24ca7a103cb408ab102718fff38ec170a6ca03e263d
```

Compare all 64 hexadecimal characters. A matching file hash establishes identity with the recorded artifact, not model compatibility or publisher authenticity by itself. Obtain files through their official project releases and independently authenticate published checksums where available.

Sources:

- xperable: <https://github.com/j4nn/xperable>
- OrionOS XDA thread: <https://xdaforums.com/t/rom-15-0-unofficial-orionos-project-v15-1-for-sony-xz2-premium-aurora.4719146/>
- OrionOS release directory: <https://sourceforge.net/projects/aurora-rom/files/orion/15.1/20250303/>
- j4nn TWRP: <https://sourceforge.net/projects/tama-testbuilds/files/aurora/twrp-3.7.1_12-0_LOCAL-20251206-01-aurora.img/download>

No exact ROM ZIP filename, TWRP checksum, or official Google apps download URL was retained in the supplied summary; none is invented here.

## Observed sequence

### 1. Prepare the matching P118 bootloader, not a complete stock downgrade

The matching KDDI package above supplied a boot manifest for `aurora`, commercial `B6B5`, P118. A boot-only staging directory was prepared on the HDD by copying the package's **intact `boot` subdirectory**, retaining its layout and package metadata. newflasher 58 flashed `bootloader_a`, including the package TA material.

This was not a flash of every file in the stock package and not a flash of an Android `boot.img`. Sony's stock `boot` directory/bootloader package and a custom ROM's `boot` partition image are different things. Do not turn this record into a full firmware flash or manually extract an arbitrary selection of bootloader components.

Readable backups of TA unit `TA084F08FD` matched the package. This unit label is not a device serial and no TA values are included here. Readable `Read-all-TA1` / `Read-all-TA2` exports were **not a complete DRM backup**. They do not prove that protected keys were captured or can be restored.

**Checkpoint:** verify the actual target is the expected P118 bootloader before exploit testing. Starting Q207 is not the target. The journal does not supply a universal newflasher prompt transcript; confirm the staging contents and selected writes before proceeding.

### 2. Use the transport that actually worked

Native Windows WinUSB, installed with official Zadig 2.9, worked for the exploit. Select only the intended phone fastboot interface; do not replace unrelated USB drivers.

Observed failures:

- The official XZ2 ADB INF did not match the fastboot interface `0fce:0dde`. ADB and fastboot are distinct interfaces/modes.
- An old Flashtool INF/catalog failed validation. **Do not disable driver-signature enforcement to work around this.**
- WSL through usbipd could run ordinary fastboot queries, but the original xperable USB transfer above 4 MiB failed with insufficient memory.
- Chunked transfers in that environment transferred data but did not produce an exploit hit. Query success and transfer completion were not exploit success.

### 3. Test and patch P118 with xperable

The successful Windows executable was built from the upstream revision listed above, without patching P118 model constants. Its `LinuxLoader-p118.pe` blob must match the recorded SHA-256. Upstream documents that options execute immediately in their supplied order; `-s` must precede the test that consumes it.

Observed sizes are **session-specific experiments, not constants for every SOV38**:

| Size | Observed result |
| --- | --- |
| `0x400f90`, `0x40af90` | Misses with default-size attempts |
| `0x430f90` | Test 3 miss |
| `0x450f90`, `0x480f90` | Crashes; not successful fixes |
| `0x440f90` | Test 3 success, distance `0x419c00`, hit `0x032274` |

After the successful test, a manual reset back to blue-LED fastboot preceded patching. At size `0x440f90`, the first `-5` attempt accepted the payload but **failed verification**. After another manual reset, the next retry verified the patch.

Illustrative command forms from upstream option semantics, **not a complete literal session transcript and not an unattended recipe**:

```powershell
.\xperable.exe -B
.\xperable.exe -s 0x440f90 -3
```

**Stop between tests.** Confirm the exact bootloader and interpret the response. A miss, crash, pipe error, or payload acceptance without verification is not permission to unlock. Reset/re-enter fastboot when required; reassess unexpected behavior instead of blindly sweeping sizes.

Only after the matching test result and a fresh, understood fastboot state, the recorded patch form was:

```powershell
.\xperable.exe -s 0x440f90 -5
```

**Checkpoint:** require a verified patch. Do not issue the next command after a failed verification. The successful session needed a reset and retry; that does not guarantee another phone will behave identically.

### 4. Request unlock, then verify persistence after reset

After patch verification, the patched command `oem unlock Y` was accepted. Its response, `Device already unlocked`, was misleading as a standalone state report; it was not authoritative proof of the final persistent state.

The corresponding command form is destructive and must require an explicit decision to unlock:

```powershell
.\xperable.exe -c "oem unlock Y" -1
```

An immediate xperable `-A` query showed `unlocked:no`, followed by a pipe error. After a manual reset back to fastboot, the standard query reported `yes`:

```powershell
fastboot getvar unlocked
```

**Checkpoint:** the post-reset standard fastboot result was the evidence of persistent unlock. Do not accept the earlier message or transient query as the final answer. No Sony unlock code was involved.

### 5. Slot A failure and temporary recovery access

Later, after boot/recovery attempts, slot A was marked unbootable with retries `0`. **The cause is unknown.** Do not attribute this conclusively to the exploit, a particular image, or a particular reboot.

The session switched to slot B, whose bootloader was Q206. Persistent unlock remained. This was a device-specific recovery decision, not an instruction to switch every reader to B.

Recovery attempts then proceeded as follows:

1. The external Orion boot image initially booted temporarily into charging mode with Stardustkernel, rather than the expected recovery UI.
2. Flashing that image to `boot_b` and rebooting recovery hung.
3. Temporarily booting the j4nn aurora TWRP image provided running root ADB recovery, despite a blank UI and an invalid panel-topology report.

Do not treat the external boot image as identical to the ROM payload's `boot` image. Their difference is normal here: the ROM payload boot image supplied the working normal recovery later.

For a compatible, independently verified recovery image on a persistently unlocked device, temporary boot uses this command shape:

```powershell
fastboot boot "<verified-aurora-recovery.img>"
```

The placeholder is not a filename to run literally. Confirm target, compatibility, and current slot before use. Temporary boot does not itself flash `boot_a` or `boot_b`.

### 6. Capture the current modified state before more writes

From the working root ADB TWRP environment on B, the session backed up **77 raw partitions**, with **12,111,523,840 verified bytes**, to HDD directory `SOV38_RawBackup_20261021`.

The inspected `backup_sov38.py` implementation:

- Is bound to one device and local host paths; it is not a portable backup utility.
- Requires root recovery and slot B, inventories named block partitions, and checks required partitions.
- Skips `userdata` and by-name aliases, avoiding duplicate alias copies.
- Checks partition byte sizes and host space; compares device SHA-256 before and after capture with the local image hash.
- Uses partial files until verification succeeds and records a completion manifest.

**Limits:** userdata and RPMB were not backed up. These images represent an already modified state, including earlier bootloader/recovery changes, not pristine stock. A raw TA image captured at this later stage does not retroactively constitute a pre-unlock DRM backup. Matching hashes prove captured-byte consistency, not completeness of device recovery or restore feasibility.

Keep raw images, manifests, recovery logs, and device-specific metadata private. The existing backup script must not be run blindly on another phone.

### 7. Synchronize selected B firmware partitions to A

Before the ROM install, the reviewed `sync_b_to_a.py` used an explicit allowlist, with exclusions equivalent to the official Lineage copy-partitions approach for the relevant categories. It was **not a wholesale clone of B** and did not execute the official copy-partitions package.

Its complete pair allowlist was:

```text
abl aop bluetooth cmnlib cmnlib64 devcfg dsp hyp keymaster modem
 oem qupfw rdimage tz tzxfl tzxflattest vendor xbl xbl_config xfl
```

`boot`, `dtbo`, `system`, `vbmeta`, and unsuffixed partitions were not copied. `vendor` was explicitly included and verified.

Observed result:

- 17 differing pairs were copied B to A, bringing the selected firmware components to the B/Q206 state.
- `dsp`, `rdimage`, and `xfl` were already identical and skipped.
- Original A and B images were backed up first. The script checked live/local hashes, sizes, resolved block paths, destination mount state, and backup completeness before writes, then verified each destination hash afterward.
- The script did **not** switch the active slot.

The inspected implementation defaults to a read-only preflight; `--apply` writes raw blocks. It rejects an existing apply audit rather than offering automatic resume. A partial run therefore requires manual reassessment, not deletion of the audit and blind retry.

**Do not blindly run this serial-bound script.** Its selection and B-to-A direction were chosen for this phone's unbootable A and known B state. Copying the wrong source, wrong slot, incompatible firmware, or mounted partitions can destroy the remaining usable path. This README intentionally does not provide a copy-and-run invocation.

### 8. Install the verified OrionOS ZIP to inactive A

The OrionOS ZIP was stored locally under `CUSTOMROM` and identified by the recorded hash. Its payload contained five images: `boot`, `dtbo`, `system`, `vbmeta`, and `vendor`. Their sizes matched this phone's partition geometry.

With B still active, TWRP installed `/tmp/orion.zip` successfully to **inactive slot A**. The summary does not retain the exact TWRP invocation; do not infer that a blank UI was usable or substitute a guessed command.

After successful installation:

```powershell
adb reboot recovery
```

Orion recovery UI worked from the updated A environment. Confirm that the newly installed recovery and intended slot are actually running before adding Google apps. The earlier external boot image's charging-mode behavior did not describe this payload boot image.

### 9. Install Google apps, format data, boot setup

The package was official MindTheGapps **Android 15, arm64**, build `20260915_032013`, with the SHA-256 recorded above. It was independently checked against the official hash in the session.

1. In the working Orion recovery, select **Apply update > Apply from ADB** (ADB sideload).
2. Sideload the verified package using the command shape below.
3. The user accepted the expected separate signature warning, error `21`, **only because the official package hash had been independently verified**. This is not permission to ignore arbitrary signature failures, corruption, wrong architecture, or unknown packages.
4. Installation finished with status `0`.
5. The user selected **Factory reset > Format data**, confirmed the destructive operation, and reported it completed.
6. Reboot reached the Hello/setup screen; the user reported success.

```powershell
adb sideload "<verified-MindTheGapps-15-arm64.zip>"
```

**Formatting data irreversibly erases user data. Do not automate or pre-approve this confirmation.** Verify installation success before formatting/rebooting, and do not interpret host progress alone as a successful recovery-side install.

Do not enable Orion's broken `fullres` mode. No working full-resolution fix was established in this session.

## Guarded replication checklist

This is a checkpoint list, not a batch file. Commands below are examples to run only when the phone is in the correct mode and the intended single device is selected. Keep output private if it includes device identifiers.

### Read-only checks

For files on Windows:

```powershell
Get-FileHash -Algorithm SHA256 -LiteralPath "<artifact-path>"
```

For fastboot, query individual fields rather than publishing an unredacted `getvar all` dump:

```powershell
fastboot getvar product
fastboot getvar version-bootloader
fastboot getvar current-slot
fastboot getvar unlocked
```

For an already running ADB recovery:

```powershell
adb get-state
adb shell id
adb shell getprop ro.boot.slot_suffix
```

Unsupported queries or contradictory results require reassessment. Do not confuse Android userspace, recovery, Sony flash mode, and bootloader fastboot.

### Go / stop gates

| Gate | Required before proceeding |
| --- | --- |
| Before bootloader preparation | Personal-data backup, stable power, exact SOV38 package/manifest match, reviewed boot-only staging, explicit flash confirmation |
| Before exploit | Confirmed P118, verified tool/blob provenance, working native transport, known reset/re-entry procedure |
| Before unlock | Verified runtime patch, explicit unlock confirmation, acceptance of possible data loss |
| Before custom images | Persistent `unlocked: yes` after reset, compatible aurora image, known active slot |
| Before partition synchronization | Usable source slot, verified private backups, exact allowlist/geometry/hash/mount checks, explicit write confirmation |
| Before ROM install | Authentic verified ZIP, matching five-image geometry, known active/inactive slots, working recovery control path |
| Before Google apps | Newly installed Orion recovery running, official Android 15 arm64 package/hash, correct sideload mode |
| Before format | Successful installs, independent user confirmation that data may be erased |
| Completion | Successful setup boot; record hardware/DRM tests separately instead of assuming success |

If your initial or intermediate state differs, this journal is not sufficient authorization to continue. Preserve the last known usable slot when possible; do not experiment on both slots simultaneously.

## Can this become a one-click tool?

**A guided state-machine tool is feasible; a safe blind one-click autorun is not established. No such program was implemented for this documentation task.**

A responsible implementation would:

- Start with read-only identification, bootloader/slot checks, file provenance and SHA-256 checks, geometry checks, storage/power checks, and a dry-run plan.
- Keep device identifiers local; redact exportable logs and never upload TA/raw backups automatically.
- Separate driver setup, Sony flash mode, native fastboot, exploit testing, recovery, backup, partition synchronization, ROM installation, Google apps, and formatting into explicit states.
- Require explicit confirmations immediately before each destructive stage. Never default to unlock, raw copying, formatting, or relocking.
- Pause for manual resets/reconnections and revalidate identity and slot afterward. Detect wrong modes, lost transports, stale state, and failed verification.
- Treat exploit sizes/offsets as observed variable parameters, not fixed SOV38 constants. A transferred payload or accepted command must not advance the state without verified success.
- Preserve audited backups and stop on partial writes. Resume only after a deliberate assessment of live state; never blindly replay a prior plan.
- Require a post-reset persistent unlock check and recovery-side install status, not just host exit codes or messages.

The existing Python files are private, device-specific session helpers, not the basis for an unattended public release without redesign and testing across devices. Recovery from arbitrary failure, EDL restoration, universal offsets, and full hardware validation remain unproven.

## Private evidence and publication hygiene

Optional local evidence resides under `O:\SonyStuff\SOV38_RawBackup_20261021`, including `manifest.json`, `inventory.txt`, recovery/kernel logs, and `sync_b_to_a.apply.json` where present. Inspect locally; do not distribute this directory. The scripts themselves also contain private device bindings and host paths.

This README deliberately contains no phone serial, IMEI, TA contents, DRM keys, or other device secrets. Public build identifiers, the TA unit label, USB VID/PID, source commits, and artifact hashes are retained because they identify procedures/artifacts rather than the individual handset. Review any separately attached screenshots or logs independently; this document does not sanitize them.
