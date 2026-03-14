# Getting Intel Wireless (WiFi + Bluetooth) Working on macOS Sequoia

A comprehensive guide based on real-world troubleshooting on macOS Sequoia 15.7.4 (build 24G517) with an Intel AC 3168NGW (USB 8087:0AA7, BT 4.2). This guide should also apply to macOS Tahoe (macOS 26) with minor additions.

**Hardware tested:** Intel i3-7020U (Kaby Lake), Intel HD 620, SMBIOS MacBookPro15,2, OpenCore 1.0.6.

---

## Table of Contents

1. [Background: What Changed in Sequoia](#background-what-changed-in-sequoia)
2. [Prerequisites](#prerequisites)
3. [Step 1: Device Spoofing (Intel as Broadcom)](#step-1-device-spoofing)
4. [Step 2: Kexts and Kernel Configuration](#step-2-kexts-and-kernel-configuration)
5. [Step 3: Security Settings](#step-3-security-settings)
6. [Step 4: Bluetooth NVRAM Keys](#step-4-bluetooth-nvram-keys)
7. [Step 5: OCLP Root Patches](#step-5-oclp-root-patches)
8. [Step 6: Disable the Spoof](#step-6-disable-the-spoof)
9. [Step 7: NVRAM Reset and Verification](#step-7-nvram-reset-and-verification)
10. [DRM Fix (Apple Music)](#drm-fix-apple-music)
11. [Tahoe-Specific Notes](#tahoe-specific-notes)
12. [What We Tried That Did NOT Work](#what-we-tried-that-did-not-work)
13. [Critical Gotcha: PlistBuddy Data Encoding Bug](#critical-gotcha-plistbuddy-data-encoding-bug)
14. [References](#references)

---

## Background: What Changed in Sequoia

In macOS Sequoia (and later), Apple made a fundamental architectural change to Bluetooth:

- **Before Sequoia:** Bluetooth USB transport was handled by the kernel-space kext `IOBluetoothHostControllerUSBTransport`.
- **Sequoia and later:** This kext was **removed**. Bluetooth USB transport moved entirely to **userspace** via `bm3_usb` inside the `bluetoothd` daemon.

This means:
- Kexts that patched kernel-space BT transport (like IntelBTPatcher) can cause **kernel panics** on Sequoia (see [GitHub issue #486](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/issues/486)).
- BlueToolFixup now patches `bluetoothd` in userspace via Lilu's `cs_validate_page` kernel hook.
- OCLP WiFi root patches can **break Bluetooth** as a side effect, requiring additional NVRAM keys to fix.

Additionally, Apple moved WiFi to the IOSkywalk framework, which requires injecting legacy WiFi kexts for Intel cards to work.

---

## Prerequisites

Download the following:

| Component | Source | Notes |
|-----------|--------|-------|
| [OpenCore Legacy Patcher (OCLP)](https://github.com/dortania/OpenCore-Legacy-Patcher) | GitHub | Standard OCLP works; OCLP-Mod (laobamac) also works |
| [IO80211FamilyLegacy.kext](https://github.com/dortania/OpenCore-Legacy-Patcher/blob/main/payloads/Kexts/Wifi/IO80211FamilyLegacy-v1.0.0.zip) | OCLP payloads | Contains AirPortBrcmNIC.kext plugin |
| [IOSkywalkFamily.kext](https://github.com/dortania/OpenCore-Legacy-Patcher/blob/main/payloads/Kexts/Wifi/IOSkywalkFamily-v1.2.0.zip) | OCLP payloads | Replaces Apple's IOSkywalk |
| [AMFIPass.kext](https://github.com/dortania/OpenCore-Legacy-Patcher/blob/main/payloads/Kexts/Acidanthera/AMFIPass-v1.4.1-RELEASE.zip) | OCLP payloads | Required for root patches |
| [AirportItlwm.kext](https://github.com/openintelwireless/itlwm/releases) | OpenIntelWireless | **Use the VENTURA build** (v2.3.0) |
| [IntelBluetoothFirmware.kext](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/releases) | OpenIntelWireless | Loads BT firmware |
| [BlueToolFixup.kext](https://github.com/acidanthera/BrcmPatchRAM/releases) | Acidanthera | Patches bluetoothd |
| [ResetNvramEntry.efi](https://github.com/acidanthera/OpenCorePkg/releases) | OpenCorePkg | For NVRAM reset from boot picker |
| [Hackintool](https://github.com/benbaker76/Hackintool) | GitHub | To find your WiFi card's PCI path |

**Important:** Use the **Ventura** build of AirportItlwm (`AirportItlwm_v2.3.0_stable_Ventura.kext.zip`), NOT the Sequoia build. The Ventura build uses the IO80211FamilyLegacy API, which is what we inject.

---

## Step 1: Device Spoofing

The key insight: we temporarily spoof the Intel card as a Broadcom BCM4360 so that OCLP will detect it and apply WiFi root patches. After patching, we disable the spoof.

### Find Your PCI Path

Open **Hackintool** > PCIe tab, find your Intel wireless card, right-click, and select **Copy Device Path**.

Alternatively, from Terminal:
```bash
ioreg -l -w0 | grep -B80 "iwm-3168" | grep "+-o.*RP\|+-o.*PXSX"
# Look for the device tree: RP##@XX,Y > PXSX@0
# Convert to: PciRoot(0x0)/Pci(0xXX,0xY)/Pci(0x0,0x0)
```

For example, `RP11@1D,2 > PXSX@0` translates to `PciRoot(0x0)/Pci(0x1D,0x2)/Pci(0x0,0x0)`.

### Add DeviceProperties Spoof

In your `config.plist`, go to `DeviceProperties > Add` and create a new dictionary with your card's PCI path. Add these properties:

| Key | Type | Value |
|-----|------|-------|
| IOName | String | pci14e4,43a0 |
| compatible | String | pci106b,117 |
| device-id | Data | A0430000 |
| device_type | String | Network Controller |
| model | String | BCM4360 802.11ac Wireless Network Adapter |
| name | String | pci14e4,43a0 |
| pci-aspm-default | Number | 0 |
| subsystem-id | Data | 17010000 |
| subsystem-vendor-id | Data | 6B100000 |
| vendor-id | Data | E4140000 |

**Command-line method** (use `plutil` for Data fields — see [PlistBuddy warning](#critical-gotcha-plistbuddy-data-encoding-bug)):

```bash
PCIPATH="PciRoot(0x0)/Pci(0x1D,0x2)/Pci(0x0,0x0)"  # Replace with YOUR path
CONFIG="/Volumes/OPENCORE/EFI/OC/config.plist"

# String/integer properties
/usr/libexec/PlistBuddy \
  -c "Add :DeviceProperties:Add:$PCIPATH dict" \
  -c "Add :DeviceProperties:Add:$PCIPATH:IOName string pci14e4,43a0" \
  -c "Add :DeviceProperties:Add:$PCIPATH:compatible string pci106b,117" \
  -c "Add :DeviceProperties:Add:$PCIPATH:device_type string 'Network Controller'" \
  -c "Add :DeviceProperties:Add:$PCIPATH:model string 'BCM4360 802.11ac Wireless Network Adapter'" \
  -c "Add :DeviceProperties:Add:$PCIPATH:name string pci14e4,43a0" \
  -c "Add :DeviceProperties:Add:$PCIPATH:pci-aspm-default integer 0" \
  "$CONFIG"

# Data properties (MUST use plutil for correct binary encoding)
plutil -insert "DeviceProperties.Add.$PCIPATH.device-id" -data "oEMAAA==" "$CONFIG"
plutil -insert "DeviceProperties.Add.$PCIPATH.subsystem-id" -data "FwEAAA==" "$CONFIG"
plutil -insert "DeviceProperties.Add.$PCIPATH.subsystem-vendor-id" -data "axAAAA==" "$CONFIG"
plutil -insert "DeviceProperties.Add.$PCIPATH.vendor-id" -data "5BQAAA==" "$CONFIG"
```

Base64 reference for the Data fields:
| Hex Value | Bytes | Base64 |
|-----------|-------|--------|
| A0430000 | 0xA0 0x43 0x00 0x00 | oEMAAA== |
| 17010000 | 0x17 0x01 0x00 0x00 | FwEAAA== |
| 6B100000 | 0x6B 0x10 0x00 0x00 | axAAAA== |
| E4140000 | 0xE4 0x14 0x00 0x00 | 5BQAAA== |

---

## Step 2: Kexts and Kernel Configuration

### Kexts to Add

Place all kexts in `EFI/OC/Kexts/` and add them to `config.plist > Kernel > Add`. The **load order matters**:

| Order | Kext | Notes |
|-------|------|-------|
| 1 | Lilu.kext | Must load first (dependency for all patches) |
| 2 | IOSkywalkFamily.kext | Replaces Apple's IOSkywalk |
| 3 | IO80211FamilyLegacy.kext | Legacy WiFi framework |
| 4 | IO80211FamilyLegacy.kext/Contents/PlugIns/AirPortBrcmNIC.kext | Plugin for BrcmNIC |
| 5 | AMFIPass.kext | Allows modified system files to load |
| 6 | AirportItlwm.kext | Intel WiFi driver (VENTURA build) |
| 7 | IntelBluetoothFirmware.kext | Loads Intel BT firmware |
| 8 | BlueToolFixup.kext | Patches bluetoothd for BT |

**Do NOT enable:**
- `IntelBTPatcher.kext` — causes kernel panics on Sequoia (`IOGMD: not wired for IODMACommand`)
- `itlwm.kext` — conflicts with AirportItlwm; use one or the other

### Block Apple's IOSkywalk

In `Kernel > Block`, add (or enable if already present):

| Key | Value |
|-----|-------|
| Identifier | com.apple.iokit.IOSkywalkFamily |
| MinKernel | 24.0.0 |
| Strategy | Exclude |
| Arch | x86_64 |
| Enabled | true |

### UEFI Drivers

Add `ResetNvramEntry.efi` to `EFI/OC/Drivers/` and `config.plist > UEFI > Drivers`:

```
Path: ResetNvramEntry.efi
Enabled: true
```

---

## Step 3: Security Settings

### SIP (csr-active-config)

Set `NVRAM > 7C436110-AB2A-4BBB-A880-FE41995C9F82 > csr-active-config` to `03080000` (0x803).

This enables:
- `CSR_ALLOW_UNTRUSTED_KEXTS` (0x001)
- `CSR_ALLOW_UNRESTRICTED_FS` (0x002)
- `CSR_ALLOW_ANY_RECOVERY_OS` (0x800)

Using plutil:
```bash
plutil -replace "NVRAM.Add.7C436110-AB2A-4BBB-A880-FE41995C9F82.csr-active-config" \
  -data "AwgAAA==" "$CONFIG"
```

### SecureBootModel

Set `Misc > Security > SecureBootModel` to `Disabled`. Required for Intel WiFi.

### AMFI

Do **NOT** add `amfi=0x80` or `amfi_get_out_of_my_way` to boot-args. Use `AMFIPass.kext` instead — it achieves the same effect without breaking other apps (notably Firefox crashes with `amfi=0x80`).

If you must use `amfi=0x80`, also add `ipc_control_port_options=0` to boot-args to prevent Firefox and other apps from crashing.

---

## Step 4: Bluetooth NVRAM Keys

OCLP WiFi root patches break Bluetooth. Fix it by adding these NVRAM keys under `NVRAM > 7C436110-AB2A-4BBB-A880-FE41995C9F82`:

| Key | Type | Value | Base64 |
|-----|------|-------|--------|
| bluetoothExternalDongleFailed | Data | 00 (1 byte) | AA== |
| bluetoothInternalControllerInfo | Data | 14 zero bytes | AAAAAAAAAAAAAAAAAAA= |

Using plutil:
```bash
plutil -replace "NVRAM.Add.7C436110-AB2A-4BBB-A880-FE41995C9F82.bluetoothExternalDongleFailed" \
  -data "AA==" "$CONFIG"
plutil -replace "NVRAM.Add.7C436110-AB2A-4BBB-A880-FE41995C9F82.bluetoothInternalControllerInfo" \
  -data "AAAAAAAAAAAAAAAAAAA=" "$CONFIG"
```

**Warning:** If using PlistBuddy to set Data fields, read the [PlistBuddy encoding bug section](#critical-gotcha-plistbuddy-data-encoding-bug) first. Getting this wrong was the single biggest time sink in our troubleshooting.

---

## Step 5: OCLP Root Patches

1. Save your config.plist and **reboot** (WiFi will NOT work yet — this is expected)
2. Open **OCLP** > **Post-Install Root Patch**
3. OCLP should detect "Broadcom BCM4360" (the spoof) and offer WiFi patches
4. Click **Start Root Patching**
5. **Do NOT reboot yet** — proceed to Step 6 first

If you get SIP-related errors, try resetting NVRAM from the OpenCore boot picker.

---

## Step 6: Disable the Spoof

After OCLP patches are applied, disable the spoof by commenting out the PCI path with a `#` prefix:

```bash
# Using PlistBuddy
/usr/libexec/PlistBuddy \
  -c "Copy :DeviceProperties:Add:$PCIPATH :DeviceProperties:Add:#$PCIPATH" \
  -c "Delete :DeviceProperties:Add:$PCIPATH" \
  "$CONFIG"
```

The key should now read `#PciRoot(0x0)/Pci(0x1D,0x2)/Pci(0x0,0x0)` — OpenCore ignores keys starting with `#`.

**Now reboot.** WiFi should work via AirportItlwm with the legacy framework patched by OCLP.

---

## Step 7: NVRAM Reset and Verification

After rebooting, if Bluetooth doesn't work immediately:

1. In the OpenCore boot picker, select **Reset NVRAM** (requires ResetNvramEntry.efi)
2. The machine reboots — select your macOS volume
3. Wait 1-2 minutes after login for BT to initialize

### Verification Commands

```bash
# Check WiFi
system_profiler SPAirPortDataType | grep -E "Card Type|Status|PHY Mode|Channel"

# Check Bluetooth
system_profiler SPBluetoothDataType

# Check BT USB device ownership
ioreg -l -w0 -r -n "Bluetooth USB Host Controller" | grep -i "UsbExclusive\|bluetoothd"
```

Expected BT output:
- **State:** On
- **Bluetooth Address:** (non-NULL, e.g., `C0:B8:83:B2:21:A3`)
- **Chipset:** Should show a value (not blank)

---

## DRM Fix (Apple Music)

After applying OCLP root patches, Apple Music DRM playback may break (tracks appear to play but produce no sound). This is a WhateverGreen/iGPU issue.

Add to boot-args:
```
unfairgva=4 revpatch=sbvmm,asset
```

| unfairgva value | Effect |
|-----------------|--------|
| 1 | Software DRM for FairPlay 1 (may partially work) |
| 3 | Software DRM + software decoder (partially worked for us) |
| **4** | **Override iGPU DRM decoder (fully fixed for us)** |

`revpatch=sbvmm,asset` is from RestrictEvents.kext and helps with macOS version checks on unsupported SMBIOS models.

---

## Tahoe-Specific Notes

For macOS 26 Tahoe, add these **additional** boot-args:

```
-amfipassbeta    # Required for AMFIPass on Tahoe (system won't boot without it)
-ibtcompatbeta   # Enables IntelBTPatcher on unsupported macOS versions
```

References from the community:
- OCLP-Mod 3.1.5 (laobamac) confirmed working on Tahoe with `-amfipassbeta`
- Alternative WiFi approach: `itlwm.kext 2.3.0` + `Heliport 2.0.0 alpha` (no root patches needed)
- rexchou's [Hackintosh_script](https://github.com/rexchou/Hackintosh_script) `pacth_wifi.sh` can be used instead of OCLP for WiFi patching

---

## What We Tried That Did NOT Work

Documenting failed approaches to save others the time:

### 1. BlueToolFixup boot-args alone
**Tried:** `-btlfxboardid -btlfxallowanyaddr -btlfxnvramcheck -btlfxdbg -btlfxbeta`
**Result:** No effect on Sequoia. BlueToolFixup's source code shows `-btlfxnvramcheck` is ignored on Sequoia (only applies to Sonoma and earlier). The `-btlfxboardid` flag is required on Sonoma+ for board-id patching but alone isn't sufficient.

### 2. IntelBTPatcher + `-ibtcompatbeta`
**Tried:** Enabled IntelBTPatcher.kext with `-ibtcompatbeta` boot-arg.
**Result:** No improvement on Sequoia. IntelBTPatcher v2.5.0 has a known kernel panic issue on Sequoia (`IOGMD: not wired for IODMACommand`, [GitHub #486](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/issues/486), 118+ comments, still open). We got lucky and didn't panic, but the kext did nothing useful.

### 3. `amfi=0x80` + SIP changes
**Tried:** Added `amfi=0x80` to boot-args with `csr-active-config: 0x803`.
**Result:** BT still didn't work. Additionally, `amfi=0x80` caused Firefox to crash on launch. Adding `ipc_control_port_options=0` fixed Firefox but BT remained broken. randomappleboi's guide explicitly says NOT to use `amfi=0x80`.

### 4. OCLP-Mod WiFi patches without BT NVRAM keys
**Tried:** Used OCLP-Mod for WiFi patching without the `bluetoothExternalDongleFailed` and `bluetoothInternalControllerInfo` NVRAM keys.
**Result:** WiFi worked, BT broken. The OCLP WiFi patches (which modify IO80211.framework, WiFiPeerToPeer.framework, and wifip2pd) break BT as a side effect.

### 5. BT NVRAM keys with wrong encoding
**Tried:** Added the correct NVRAM keys but using PlistBuddy's `data` type, which stored them as ASCII text instead of binary.
**Result:** No effect — `bluetoothd` read garbage values. This was our biggest mistake and time sink. See below.

### 6. Wrong csr-active-config value (Claude's mistake.)
**Tried:** Set `csr-active-config` to `00380000` based on a WebFetch summary.
**Result:** Completely wrong value. `00380000` = 0x3800 (wrong SIP flags). The correct value is `03080000` = 0x803. Always verify against the original source, not AI summaries.

---

## Critical Gotcha: PlistBuddy Data Encoding Bug

This was the single most time-consuming mistake in our troubleshooting. **`PlistBuddy` does NOT correctly handle hex Data fields.**

When you run:
```bash
/usr/libexec/PlistBuddy -c "Add :Key data 03080000" file.plist
```

You might expect it stores bytes `0x03 0x08 0x00 0x00`. **It does NOT.** It stores the ASCII string "03080000" as bytes:
```
0x30 0x33 0x30 0x38 0x30 0x30 0x30 0x30
```

Which in base64 is `MDMwODAwMDA=` instead of the correct `AwgAAA==`.

### How to verify

```bash
# Check what's actually stored (shows base64)
plutil -extract "NVRAM.Add.7C436110-AB2A-4BBB-A880-FE41995C9F82.csr-active-config" raw -o - config.plist
```

- `AwgAAA==` = correct (binary 03 08 00 00)
- `MDMwODAwMDA=` = WRONG (ASCII text "03080000")

### The fix: Always use `plutil` for Data fields

```bash
# CORRECT way to set Data fields
plutil -replace "NVRAM.Add.7C436110-AB2A-4BBB-A880-FE41995C9F82.csr-active-config" \
  -data "AwgAAA==" config.plist

# Or use -insert for new keys
plutil -insert "Key.Path" -data "BASE64VALUE" config.plist
```

### Quick base64 conversion

To convert hex to base64 for plutil:
```bash
echo "03080000" | xxd -r -p | base64
# Output: AwgAAA==
```

---

## Final Working Boot-Args

```
-v keepsyms=1 debug=0x100 alcid=3 -btlfxallowanyaddr -btlfxnvramcheck -btlfxboardid -btlfxdbg -btlfxbeta -lilubetaall unfairgva=4 revpatch=sbvmm,asset -no_compat_check
```

For Tahoe, add: `-amfipassbeta -ibtcompatbeta`

---

## References

- [randomappleboi: Native WiFi for Hackintoshes with Intel Wireless cards on macOS Sequoia](https://github.com/randomappleboi/Native-Wifi-for-Hackintoshes-with-Intel-Wireless-cards-on-macOS-sequoia) — The guide that ultimately worked. Covers spoofing, OCLP patching, and BT NVRAM fix.
- [rexchou: Hackintosh_script](https://github.com/rexchou/Hackintosh_script) — Dell Vostro 5490 Hackintosh with Intel WiFi/BT working on Tahoe. Includes `pacth_wifi.sh` for manual WiFi patching.
- [OpenIntelWireless FAQ](https://openintelwireless.github.io/itlwm/FAQ.html) — Official FAQ for AirportItlwm/itlwm.
- [IntelBTPatcher Issue #486](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/issues/486) — Kernel panic on Sequoia, still open.
- [BlueToolFixup Source (BrcmPatchRAM)](https://github.com/acidanthera/BrcmPatchRAM) — Source code analysis revealed Sonoma+ board-id behavior and Sequoia-specific patches.
- [Dortania's OpenCore Install Guide](https://dortania.github.io/OpenCore-Install-Guide/) — General Hackintosh reference.

---

*Last updated: 2026-03-14. Tested on macOS Sequoia 15.7.4, OpenCore 1.0.6, Intel AC 3168NGW.*
