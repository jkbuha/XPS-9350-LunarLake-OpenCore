# Dell XPS 13 9350 (Lunar Lake) macOS Tahoe with OpenCore

<!-- Add a screenshot as ./screenshot/screen.jpg -->

> **Experimental.** Lunar Lake graphics acceleration on macOS is new and still has an open
> hard-freeze bug (see [Known issues](#known-issues)). Use this EFI on a machine you can recover.

# Details

| OpenCore Version | 1.0.8 |
| --- | --- |
| macOS Version | 26.7.3 (Tahoe) |
| SMBios | MacBookAir9,1 |
| iGPU driver | NMXGPU (Linux `xe` on XNU) + [MeowXE3](https://github.com/cuadra8998/MeowXE3) accelerator |

# Hardware Specifications

| Hardware | Specification | Status |
| --- | --- | --- |
| CPU | Intel Core Ultra 7 256V (Lunar Lake, 4P + 4 LP-E) | ✅ Working (XCPM, CPUFriend, CpuTopologyRebuild) |
| RAM | LPDDR5X 16GB (on package) | ✅ Working |
| GPU | Intel Arc 140V (Xe2, `8086:64A0`) | ⚠️ Accelerated (Metal 3), experimental: see Known issues |
| Display | 13.4" 1920 x 1200 eDP | ✅ Working |
| Audio | Lunar Lake built-in audio | ❌ Not Working |
| WiFi | Intel BE201 (Wi-Fi 7, CNVi) | ✅ Working (AirportItlwm BE201 build) |
| Bluetooth | Intel BE201 | ✅ Working (BE201BluetoothCompat) |
| Keyboard | - | ✅ Working (VoodooPS2) |
| Trackpad | I2C HID touchpad | ✅ Working (VoodooI2C) |
| Battery | - | ✅ Working (ECEnabler, SMCBatteryManager) |
| USB-C / Thunderbolt 4 | 2 ports | ⚠️ Not fully tested (USBToolBox map) |
| Webcam | MIPI (IPU7) camera | ❌ Not Working |
| Fingerprint Sensor | - | ❌ Not Working |
| Sleep | Lunar Lake has no S3 (Modern Standby only) | ❌ Not available |
| Hibernation | `hibernatemode 25` | ✅ Working (with the MeowXE3 accelerator wake fix) |

# Overview

The XPS 13 9350 runs macOS Tahoe with **Xe2 graphics acceleration**, through two out-of-tree pieces:

1. **NMXGPU**: NullMoth's port of the Linux `xe` driver to XNU, built for Lunar Lake with the MeowXE3 patch set.
2. **MeowXE3** ([cuadra8998/MeowXE3](https://github.com/cuadra8998/MeowXE3)): the macOS accelerator (`MeowXE3Accel`),
   the display glue (`MeowXE3AGDC`) and `MeowXE3Boot.kext`, which is in this EFI.

NMXGPU, MeowXE3Accel and MeowXE3AGDC are **not** in this EFI: on macOS 26 they depend on IOGraphicsFamily, which lives
only in the System KC, so OpenCore cannot inject them. Install them into `/Library/Extensions` and approve them in
**System Settings → Privacy & Security** (every new build needs approving again). Follow the MeowXE3 README.

Without those kexts the machine boots on the firmware framebuffer (no acceleration), which is also the recovery path.

# Boot-args

The boot-args include verbose debugging flags (`-liludbgall`, `-dbgenhiolog`, `-dbgenhdbg`, `debug=0x100`) because the
graphics driver is still being debugged; they are safe to remove for daily use.

| Boot-arg | Purpose |
| --- | --- |
| `nmxintel=1` | Enables NMXGPU's Intel (Lunar Lake) path |
| `nmxgthb=7200` | Debug: NMXGPU writes a GPU power heartbeat to NVRAM for up to 7200 s after boot (remove for daily use) |
| `nmxlnlawake=N` (optional) | GPU tiles kept out of RC6: `3` (default) both, `2` media only, `0` none. Lower values save a lot of idle power but are still being tested |
| `gfxrst=4`, `agdpmod=pikera`, `-wegswitchgpu` | WhateverGreen display handling |
| `hbfx-ahbm=65` | HibernationFixup auto-hibernate mode |
| `rtcfx_exclude=80-FF` | RTCMemoryFixup: keeps macOS out of the RTC range the firmware uses (hibernation) |
| `amfi=0x80`, `-no_compat_check`, `ipc_control_port_options=0` | Needed for the unsigned driver stack and the SMBIOS |

# Hibernation

Real S3 sleep does not exist on Lunar Lake, so suspend is done as hibernation (S4):

```
sudo pmset -a hibernatemode 25
```

`SSDT-S3-9350`, HibernationFixup, RTCMemoryFixup and a ReservedMemory entry make S4 work. The display comes back
accelerated with the MeowXE3 accelerator's hibernation wake fix (MeowXE3 issue #1).

# Known issues

- **Random hard freezes** with the GPU driver loaded (the screen stops, the machine needs a forced power-off). Being
  investigated; the leading lead is copy-engine (BCS) traffic.
- **Restart after approving kexts** sometimes hangs with the backlight at full brightness. Force off and boot again.
- **Idle power** is higher than it should be while the GPU tiles are kept awake (`nmxlnlawake`, above).
- Wi-Fi only connects after login.

# SMBIOS

`PlatformInfo → Generic` has its serial number, MLB, system UUID and ROM cleared. Generate your own
(MacBookAir9,1) with [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) before use.

# Credits

- [Acidanthera](https://github.com/acidanthera) for OpenCore, Lilu, WhateverGreen, VirtualSMC, AppleALC and friends
- NullMoth for NMXGPU, ericek111 for the Xe2/Xe3 patches, [cuadra8998](https://github.com/cuadra8998) for MeowXE3
- [OpenIntelWireless](https://github.com/OpenIntelWireless) for AirportItlwm
- [VoodooI2C](https://github.com/VoodooI2C) and [VoodooPS2](https://github.com/acidanthera/VoodooPS2)
