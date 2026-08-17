# ASUS Zenbook 14 UX3405CA — CS35L41 VSPK quirk

**Author:** Gemayel Lira (<gemayellira@gmail.com>)

Hardware: ACPI HID `CSC3551`, subsystem ID `1043:1A63`, SSDT `SPKRAMPS`.

Upstream change: [docs/upstream/0001-ALSA-hda-cs35l41-Enable-VSPK-on-UX3405CA-when-ACPI-leaves-GPIO1-unused.patch](docs/upstream/0001-ALSA-hda-cs35l41-Enable-VSPK-on-UX3405CA-when-ACPI-leaves-GPIO1-unused.patch)
(`cs35l41_hda_property.c`, `missing_speaker_id_gpio2`). Cover:
[docs/upstream/COVER.txt](docs/upstream/COVER.txt).

Cirrus asked for a public ticket after `[PATCH v2]`:
[bug 221896](https://bugzilla.kernel.org/show_bug.cgi?id=221896)
(ACPI dump, stock dmesg with `VSPK: 0` on the right amp). Not merged yet.

The `module/` tree is a tested out-of-tree workaround (VSPK quirk plus
local gain/resume extras). It is not the mailing-list patch.

## Problem

Stock `missing_speaker_id_gpio2` (Stefan Binding, 2024) already parses
this `_DSD` and adds speaker-id at CRS index 2. ACPI still leaves
GPIO1 unused on the right amp (`cirrus,gpio1-func = 0`), so that
channel binds with `VSPK: 0` and volume drops about one second after
playback.

## ACPI (extracted with `acpidump` + `iasl`)

`_CRS` GPIO resources:

| Index | Pin | Direction | Role |
|---|---|---|---|
| 0 | `0x0048` | output | unused by `_DSD` |
| 1 | `0x00D0` | output | reset (shared) |
| 2 | `0x00CE` | input | speaker-id (not in `_DSD`) |
| 3 | `0x00CF` | input + `GpioInt` | IRQ |

`_DSD`:

```
cirrus,dev-index:          0, 1
reset-gpios:               SPK1, 1, 0, 0  (twice — shared reset)
cirrus,speaker-position:   0, 1           (L, R)
cirrus,boost-type:         1, 1           (external)
cirrus,gpio1-func:         1, 0           (VSPK, unused)
cirrus,gpio2-func:         2, 2           (IRQ, IRQ)
```

The volume bug is `gpio1-func = 0` on amp 1. Missing `spk-id-gpios`
is already handled upstream.

## Approach

After `cs35l41_hda_parse_acpi()`, if GPIO1 was left unused, set it to
`VSPK_SWITCH`. Do not map `10431A63` to `generic_dsd_config`: Binding
kept the `_DSD` so the correct `spk-prot-10431a63` firmware loads. A
full override produced `SPKID: -19` and, after some S3 resumes,
`PM: failed to resume: error -110` on amp `.1` then SPI `-16 (EBUSY)`.

## Tested

Machine: ASUS Zenbook 14 UX3405CA (`zion`).
Kernel: `7.0.0-27-generic` (Ubuntu), module from `updates/`
`srcversion` `955693DEB4FC9A01D9EF0EC`.

| Check | Result |
|---|---|
| Both amps bound | yes, L and R |
| VSPK | 1 on both (right was unused in ACPI) |
| SPKID | 1 (was `-19` with `generic_dsd_config`) |
| Firmware | official `spk-prot-10431a63-spkid1-{l,r}0.bin` v0.65.0 |
| Volume drop after ~1 s | gone (playback on Speaker sink) |
| `-110` / `-16` this boot | none |
| One S3 (`PM: suspend entry (deep)` 16:54) | both amps reloaded firmware; no resume error |

Before (override `generic_dsd_config`, boots 2026-08-11..14):

```
CS35L41 Bound - SSID: 10431A63, BST: 1, VSPK: 1, CH: L, FW EN: 1, SPKID: -19
CS35L41 Bound - SSID: 10431A63, BST: 1, VSPK: 1, CH: R, FW EN: 1, SPKID: -19
cs35l41-hda spi0-CSC3551:00-cs35l41-hda.1: PM: failed to resume: error -110
```

After (boot 2026-08-15 16:48):

```
cs35l41-hda spi0-CSC3551:00-cs35l41-hda.1: UX3405CA: enabling VSPK switch on GPIO1 (ACPI left it unused)
cs35l41-hda spi0-CSC3551:00-cs35l41-hda.1: Reset line busy, assuming shared reset
cs35l41-hda spi0-CSC3551:00-cs35l41-hda.0: Firmware Loaded - Type: spk-prot, Gain: 19
cs35l41-hda spi0-CSC3551:00-cs35l41-hda.0: CS35L41 Bound - SSID: 10431A63, BST: 1, VSPK: 1, CH: L, FW EN: 1, SPKID: 1
cs35l41-hda spi0-CSC3551:00-cs35l41-hda.1: Firmware Loaded - Type: spk-prot, Gain: 19
cs35l41-hda spi0-CSC3551:00-cs35l41-hda.1: CS35L41 Bound - SSID: 10431A63, BST: 1, VSPK: 1, CH: R, FW EN: 1, SPKID: 1
DSP1: cirrus/cs35l41-dsp1-spk-prot-10431a63-spkid1-l0.bin (v1): v0.65.0
DSP1: cirrus/cs35l41-dsp1-spk-prot-10431a63-spkid1-r0.bin (v1): v0.65.0
```

The `UX3405CA: enabling VSPK...` line is from the out-of-tree
`dev_info`. The upstream patch is silent and only changes GPIO1.

One suspend this boot (not a 3-cycle claim):

```
Aug 15 16:54:03 zion kernel: PM: suspend entry (deep)
Aug 15 16:54:16 zion kernel: PM: suspend exit
```

Both amps loaded firmware again at 16:54:16. No `-110` / `-16`.

## Until this is in your distro kernel

```bash
sudo apt install build-essential linux-headers-$(uname -r)
cd module
make
sudo mkdir -p /lib/modules/$(uname -r)/updates
sudo cp snd-hda-scodec-cs35l41*.ko /lib/modules/$(uname -r)/updates/
sudo depmod -a
```

Reboot, then `dmesg | grep cs35l41`. Expect `VSPK: 1`, `SPKID: 1`, and
no `failed to resume: error -110`.

Rollback:

```bash
sudo rm /lib/modules/$(uname -r)/updates/snd-hda-scodec-cs35l41*.ko
sudo depmod -a
```

Then reboot.

## Not in the mailing-list patch

- Default PCM gain 20.5 dB / `custom_amp_gain` (local preference)
- Skip software reset after shared hardware reset — already posted by
  Zhang Heng; see [bug 221161](https://bugzilla.kernel.org/show_bug.cgi?id=221161)
