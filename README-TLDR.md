# [SOV38][XZ2 Premium] Successful xperable bootloader unlock and OrionOS boot — case report, not a universal guide

Hi everyone,

I successfully unlocked the bootloader on my Japanese au/KDDI Xperia XZ2 Premium SOV38, which originally reported **Bootloader unlock allowed: No**, and installed OrionOS Android 15. The phone reached the Android setup screen.

This post documents one handset and the problems encountered along the way. It is **not a guaranteed-safe procedure**, and several parts need independent testing and maintainer review.

## Important warnings

- Bootloader and firmware writes can permanently brick the device.
- Fastboot or Sony flash mode is not guaranteed to remain available after a failed boot-chain modification. EDL recovery may require unavailable authorization.
- Backups are useful only if a suitable restoration path remains available.
- Do not copy someone else's TA, persist, modem calibration, keystore, or other device-unique partitions.
- Do not relock the bootloader while running this custom ROM.
- The exploit size used below is an observation from this session, **not a universal SOV38 parameter**.

## Starting device

| Item | Observed value |
|---|---|
| Model | Sony Xperia XZ2 Premium SOV38 |
| Carrier / region | au/KDDI, Japan |
| Device family | Tama / aurora |
| Stock Android | Android 10 |
| Stock build | `52.1.C.0.268` |
| Security patch | `2020-07-01` |
| Initial bootloader | `1310-7079_X_Boot_SDM845_LA2.0.1_Q_207` |
| Initial active slot | A |
| Initial bootloader lock | Locked |
| Service-menu unlock eligibility | No |

No phone serial number or IMEI is included here.

## What actually succeeded

1. A model-matched SOV38 P118 bootloader package was installed to slot A.
2. Unmodified upstream xperable obtained a verified RAM patch.
3. The exploit's persistent unlock command was accepted.
4. After restarting, standard fastboot returned:

```text
unlocked: yes
```

5. The phone subsequently ran Q206 from slot B while remaining unlocked.
6. After additional firmware preparation, OrionOS installed successfully and reached Android setup.

**This verifies actual bootloader unlocking.** I did not independently verify that the service-menu eligibility text itself changed from No to Yes. Those are different states.

## Sources and tools

- xperable: https://github.com/j4nn/xperable
- Author's xperable thread: https://xdaforums.com/t/xz1c-xz1-xzp-xperable-xperia-abl-fastboot-exploit-cve-2021-1931.4771931/
- SOV38 discussion, including maintainer cautions about fixed offsets and proposed patches: https://github.com/j4nn/xperable/issues/2
- newflasher v58: https://github.com/munjeni/newflasher/tree/58
- Zadig official site: https://zadig.akeo.ie/
- j4nn's Tama TWRP thread: https://xdaforums.com/t/recovery-twrp-3-7-1_12-xz2-xz2c-xz2p-xz3.4772176/
- Aurora TWRP files: https://sourceforge.net/projects/tama-testbuilds/files/aurora/
- OrionOS thread: https://xdaforums.com/t/rom-15-0-unofficial-orionos-project-v15-1-for-sony-xz2-premium-aurora.4719146/
- OrionOS release used: https://sourceforge.net/projects/aurora-rom/files/orion/15.1/20250303/
- MindTheGapps Android 15 ARM64: https://github.com/MindTheGapps/15.0.0-arm64/releases

Exact source revisions used:

```text
xperable:  0bf7060a966f91ed6251386bb6281648e416a03a
newflasher: 2da5384c71df0fdf924bf3a5ad275f6bef59ce44
pe-parse:   31ac5966503689d5693cd9fb520bd525a8710e17
```

The successful Windows xperable executable was built for P118 from upstream source without applying the disputed SOV38 patch from issue #2.

## Firmware preparation

The stock package available was:

```text
SOV38_KDDI_JP_52.0.C.5.295-R13B
```

Its boot-delivery manifest identified Aurora, P118, and the commercial B6B5 image. The device's reported root hash and security state matched the commercial configuration. The phone's firmware history also showed that it previously ran this stock version.

**These matches reduced uncertainty; they did not prove downgrade safety or anti-rollback compatibility.**

Only the intact `boot` directory was staged for newflasher—not the complete stock firmware. The operation wrote the package's boot-configuration TA units `0x084F` and `0x08FD`, then `bootloader_a`. Those two TA units had been read and preserved first, and matched the package's values.

Afterward, standard fastboot reported:

```text
version-bootloader: 1310-7079_X_Boot_SDM845_LA2.0_P_118
unlocked: no
```

Do not confuse this bootloader package with Android's `boot.img` kernel/recovery partition.

## Host USB issues

The official XZ2 Windows ADB driver package did not contain the fastboot USB ID `0FCE:0DDE`. An older Flashtool driver package was rejected because its INF did not match its signed catalog. Signature enforcement was not disabled.

WSL2 plus usbipd allowed ordinary fastboot queries, but the full exploit transfer failed with `Insufficient memory`. Splitting the transfer into smaller chunks avoided that host error but did not obtain a hit.

Native Windows with a device-specific WinUSB binding installed through Zadig allowed the full transfer. Only the Xperia fastboot interface was selected—not another USB device.

This does not establish that every WSL configuration fails or every Windows configuration works.

## xperable observations

Several sizes produced ordinary DATA/filler/OKAY responses with no hit. Others caused pipe errors, disconnects, or hangs requiring manual return to fastboot.

**USB errors alone were not treated as evidence of code execution.**

In this session, the following diagnostic produced a structured successful response:

```text
xperable.exe -v -B -U -s 0x440f90 -3

[+] test3 succeeded: distance = 0x419c00, hit from 0x032274, base = 0x988be000
```

After a manual reset into fastboot, the corresponding RAM-patch attempt was:

```text
xperable.exe -v -B -U -s 0x440f90 -5
```

The first patch attempt failed after payload upload. After another reset, a repeat produced:

```text
[+] LinuxLoader @ 0x988c2000 patched successfully
```

Only after confirmed patching was the persistent unlock command sent:

```text
xperable.exe -v -c "oem unlock Y" -1
```

It returned:

```text
OKAYDevice already unlocked
```

That message is documented behavior of the patched Tama command; it does not mean the phone was already unlocked before the attempt.

An immediate query misleadingly reported `unlocked:no` before USB failed. **The decisive check was standard fastboot after a fresh restart**, which returned `unlocked: yes`.

These commands are a record of what worked, not instructions to reuse this size blindly. Tama memory layout varies across boots. Consult the author's README before probing, and stop when the observed state differs.

## Recovery and slot complications

This was not a clean one-click progression:

- Temporarily booting Orion's supplied recovery image initially entered charger mode with the Stardust kernel rather than recovery.
- After subsequent boot/recovery attempts, slot A was marked unbootable with zero retries; the phone was then on slot B with Q206. The exact cause of the slot change was not established.
- Unlocking persisted on Q206. This demonstrates that P118 was used to perform the unlock, not that the phone must remain on P118 afterward.
- Flashing Orion's recovery to `boot_b` and requesting recovery initially left the phone at the Sony logo without ADB.
- Temporarily booting `twrp-3.7.1_12-0_LOCAL-20251206-01-aurora.img` gave root ADB access in recovery, although its UI was blank.
- TWRP logs reported invalid panel topology and DRM display errors. The precise cause was not proven.

The separate Orion recovery `boot.img` differed from the boot image contained in the ROM payload. That difference alone was not evidence of corruption; the separate image contained recovery.

## Backups and firmware consistency

With root recovery available, 77 raw partitions were backed up, totaling **12,111,523,840 bytes**, with lengths and SHA-256 hashes checked against the device. Userdata and RPMB were excluded.

These backups captured the **already-modified state**, including P118 on A and Orion recovery in `boot_b`. They were not pristine factory images or a guaranteed unbrick solution.

The Orion payload updates:

```text
boot
 dtbo
 system
 vbmeta
 vendor
```

It does not update the Sony bootloader firmware. Installing from B normally targets inactive slot A, where older firmware remained.

The official LineageOS copy-partitions utility was studied as a reference for A/B firmware preparation. **LineageOS was not installed.** A guarded, device-specific procedure was used to copy selected firmware pairs from B to A, validating backups, sizes, mappings, and hashes before writes and destination hashes afterward.

Seventeen differing pairs were copied; three identical pairs were skipped. `boot`, `dtbo`, `system`, `vbmeta`, and unslotted device-unique partitions were not copied. The active slot was not changed by this procedure.

**Do not blindly repeat this synchronization or run my device-bound script.** It addressed this handset's observed mixed-slot condition. Another phone may need different preparation or none at all.

## Orion installation and first boot

Files used:

```text
OrionOS-15.1-Hydroxide-aurora-Unofficial-Vanilla-20250303.zip
boot.img from the same release directory
MindTheGapps-15.0.0-arm64-20260915_032013.zip
```

The Orion ZIP passed archive checks and its payload fitted the relevant partitions. Its hash was checked again after transfer to recovery.

TWRP's A/B installer applied Orion to inactive slot A and update-engine reported:

```text
Update successfully applied, waiting to reboot.
```

After rebooting into the updated slot's recovery, **Orion recovery displayed correctly**. From its Apply update / Apply from ADB menu, the Android 15 ARM64 MindTheGapps package was sideloaded.

Recovery displayed a signature warning for the independently signed add-on. The download had already been checked against its official published hash; installation then completed with status 0. This is not advice to ignore signature warnings on unknown packages.

A recovery factory reset/format-data completed, followed by rebooting the system. **The phone reached the Android Hello/setup screen.**

The Orion author warns that enabling full 2160×3840 resolution breaks the UI layout. I did not enable it.

## Artifact hashes

SHA-256 identifies the tested bytes; it does not establish universal compatibility or replace publisher authentication.

```text
P118 commercial B6B5 SIN
f32baf6e9cf49c81bf8242c45916bac0d26fd6d524c0a41784649191bf378462

Extracted LinuxLoader-p118.pe
14fb6ddeef4086f76145ae25a1987f7f9af5243f3edf47efadcd22702220ed3a

Orion ROM ZIP
5a5f2183fe1bee06874c4c2f484dfffeb062c60d24764a9e50282b78863449f3

Separate Orion recovery boot.img
3633a28be82f3c8dc9b859646ab1a82299d25e176495b6804d69715535f0e2de

j4nn Aurora TWRP image
f2d6273ee3cb1a74005d75b931146da5648181fb959a9a16a5e07e1c14fd45b7

MindTheGapps ZIP
e979601ef70b03d9214ea24ca7a103cb408ab102718fff38ec170a6ca03e263d
```

## Testing status

| Feature | Result |
|---|---|
| Persistent bootloader unlock | Verified after restart |
| Orion installer result | Successful |
| Google apps installer result | Status 0 |
| Android setup screen | Reached |
| Play Store sign-in / app downloads | [Add test result] |
| Calls, SMS, mobile data, VoLTE | [Add test result] |
| Wi-Fi, Bluetooth, GPS | [Add test result] |
| Cameras, fingerprint, audio | [Add test result] |
| NFC tag reading | [Add test result] |
| FeliCa / Osaifu-Keitai payments | [Add test result; do not infer from ordinary NFC] |
| Widevine / Netflix playback quality | [Add test result] |
| Charging, battery life, temperatures | [Add test result] |
| Repeated boots / daily-use stability | [Add duration and findings] |

xperable is designed to preserve Sony DRM keys, but this report does not independently verify DRM preservation, Widevine L1, Netflix HD, or payment-service eligibility.

## Questions for maintainers and other SOV38 owners

- Has anyone reproduced this starting from SOV38 `52.1.C.0.268` / Q207?
- Is there a better-supported recovery route for SOV38 that avoids the blank Aurora TWRP display?
- Is there a documented SOV38 firmware baseline recommended for this Orion build?
- Are there corrections to the firmware-consistency procedure or interpretation of the slot failures?
- Can additional owners report NFC, cameras, calls, and long-term stability with exact build details?

Please share sanitized logs and version information—not raw TA/persist/modem backups, IMEI numbers, or serial numbers.

## Credits

Thanks to j4nn for xperable and the recovery work, munjeni for newflasher, the Orion maintainer for the ROM, and the MindTheGapps, libusb, Zadig, and LineageOS contributors whose tools or documentation supported this investigation.

This investigation used AI-assisted source review and tooling. Maintainer review and independent replication are welcome; explanations and scripts should not be treated as authoritative merely because this handset booted successfully.

---

Before publishing: replace testing placeholders, recheck artifact hashes against retained files, attach only reviewed/sanitized logs, and do not upload the complete project or backup directories.
