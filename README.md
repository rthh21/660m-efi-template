# OpenCore EFI template: Radeon 660M laptops (Rembrandt)

A starting-point OpenCore 1.0.8 EFI for laptops with a Rembrandt APU and a
Radeon 660M iGPU (Ryzen 5 6600H/HS, Ryzen 5 7535HS…), targeting macOS Sequoia
and Tahoe.

The Radeon 680M (Ryzen 7 6800H/HS, Ryzen 7 7735HS…) is the same chip
(`1002:1681`) and uses the same EFI. Only the CPU core count differs (see below).

It was derived from a config that boots macOS Tahoe on a different Rembrandt
laptop, with every machine-specific part removed. **It has not been tested on
a ThinkBook.** Treat it as a base to adapt, not a finished EFI.

## Before the first boot

1. **Generate your own SMBIOS.** `PlatformInfo > Generic` has empty `MLB`,
   `SystemSerialNumber`, `SystemUUID` and `ROM`. Run
   [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) for `iMac20,1` and fill
   them in (`ROM` = your network card's MAC address as 6 bytes).
2. **BIOS:** Secure Boot off. If your model has a graphics mode option
   (Hybrid / UMA / iGPU only), choose the iGPU-only one.
3. Check the core count (below). The EFI is set for 6 cores. An 8-core CPU
   (Ryzen 7, Radeon 680M) needs one byte changed.

## Graphics: read this if you get a black screen

**No GPU acceleration driver is included.** macOS has no driver for the
Rembrandt iGPU, so with this EFI it runs on the firmware framebuffer: you get
a picture, but no Metal acceleration, a single display and laggy animations.

Common causes of a black screen on these machines:

- **A GPU kext that does not support Rembrandt.** NootedRed and WhateverGreen
  do not support the 660M/680M. Remove them and their `DeviceProperties`
  (`device-id`, `ATY,*` etc.).
- **The panel is wired to a discrete GPU.** On switchable-graphics models the
  internal panel can be driven by the dGPU, which macOS cannot use. Set the
  BIOS to iGPU-only if the option exists.
- **Where it stops.** `-v` is already in `boot-args`, so the boot is verbose.
  OpenCore also writes a log (`opencore-*.txt`) to the root of the EFI
  partition (`Misc > Debug > Target = 67`). Photograph the last lines of
  verbose output and keep that log; they say where the boot stops.

Once the system boots, remove `-v` and set `Target` to `0`.

## What is included

| Area | Component |
| --- | --- |
| Bootloader | OpenCore 1.0.8 (`OpenRuntime`, `ResetNvramEntry`, `HfsPlus`, `OpenShell`) |
| CPU | AMD Vanilla kernel patches, `AmdTscSync`, `AMDRyzenCPUPowerManagement`, `SMCAMDProcessor`, `AppleMCEReporterDisabler`, `RestrictEvents` |
| SMC / battery | `VirtualSMC`, `SMCBatteryManager`, `ECEnabler` |
| Lenovo | `YogaSMC` (VPC: Fn keys, battery conservation) |
| Keyboard | `VoodooPS2Controller` (keyboard only; mouse/trackpad plugins disabled) |
| Touchpad | `VoodooI2C` + `VoodooI2CHID` |
| Audio | `AppleALC` + `SimpleHDA` (AppleHDA on Tahoe), `layout-id` 101 |
| USB | `USBToolBox` + `UTBDefault` |
| Bluetooth | `BlueToolFixup` |
| ACPI | `SSDT-EC`, `SSDT-USBX`, `SSDT-PLUG-ALT`, `SSDT-PNLF-AMD`, `SSDT-ALS0` |

Every binary is the unmodified official release, listed under Credits.

## What you will probably need to adapt

- **CPU core count.** The first four `Kernel > Patch` entries
  (`cpuid_cores_per_package`) are set to **6 cores** (`Replace` byte 2 = `06`),
  for the 660M CPUs. For an 8-core CPU such as the Ryzen 7 7735HS, change it
  to `08` in all four entries.
- **ACPI.** `SSDT-EC` assumes `\_SB.PCI0.LPC0.EC0`, and `SSDT-PLUG-ALT`
  defines `C000`–`C00F`. If your DSDT differs, regenerate both
  with [SSDTTime](https://github.com/corpnewt/SSDTTime) from your own DSDT.
  No HPET/IRQ fix is included; add one with SSDTTime (`FixHPET`) if audio
  needs it.
- **USB.** `UTBDefault` is a generic map. Make your own with
  [USBToolBox](https://github.com/USBToolBox/tool) and replace it with the
  generated `UTBMap.kext`.
- **Audio.** `layout-id` 101 (and `alcid=101`) suits an ALC257. If you get no
  sound, check your codec and try the layouts AppleALC lists for it.
- **Wi-Fi: not included.** It depends on your card:
  [itlwm / AirportItlwm](https://github.com/OpenIntelWireless/itlwm) for
  Intel, AirportMtkWlan for MediaTek MT7921/MT7922. Realtek cards have no
  macOS driver.
- **Bluetooth.** Only `BlueToolFixup` is included. Intel cards need
  IntelBluetoothFirmware; MediaTek cards need a firmware loader such as
  SimpleMtkBtFw.
- **Touchpad.** If the I2C touchpad does not work, it may need GPIO pinning
  or polling mode; see the VoodooI2C documentation.
- **SIP** is disabled (`csr-active-config` = `EF070000`).

## Credits

| Component | Version | Source |
| --- | --- | --- |
| OpenCore | 1.0.8 | https://github.com/acidanthera/OpenCorePkg |
| HfsPlus.efi | — | https://github.com/acidanthera/OcBinaryData |
| Lilu | 1.7.2 | https://github.com/acidanthera/Lilu |
| VirtualSMC, SMCBatteryManager | 1.3.8 | https://github.com/acidanthera/VirtualSMC |
| AppleALC | 1.9.8 | https://github.com/acidanthera/AppleALC |
| RestrictEvents | 1.1.6 | https://github.com/acidanthera/RestrictEvents |
| VoodooPS2Controller | 2.3.7 | https://github.com/acidanthera/VoodooPS2 |
| BlueToolFixup | 2.7.2 | https://github.com/acidanthera/BrcmPatchRAM |
| ECEnabler | 1.0.6 | https://github.com/1Revenger1/ECEnabler |
| VoodooI2C, VoodooI2CHID | 2.9.1 | https://github.com/VoodooI2C/VoodooI2C |
| YogaSMC | 1.5.3 | https://github.com/zhen-zen/YogaSMC |
| USBToolBox, UTBDefault | 1.2.0 | https://github.com/USBToolBox/kext |
| AMDRyzenCPUPowerManagement, SMCAMDProcessor | 0.7.2f1 | https://github.com/trulyspinach/SMCAMDProcessor |
| AmdTscSync | 2.0.0 | https://github.com/naveenkrdy/AmdTscSync |
| SimpleHDA | 1.0.1 | https://github.com/laobamac/SimpleHDA |
| AppleMCEReporterDisabler | — | XLNC (codeless kext) |
| AMD Vanilla patches | — | https://github.com/AMD-OSX/AMD_Vanilla |
| SSDTs | — | generated with SSDTTime / Dortania samples |

Each component keeps its own license.
