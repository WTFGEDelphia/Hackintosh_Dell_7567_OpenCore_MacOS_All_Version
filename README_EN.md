# Hackintosh Dell 7567 OpenCore

[![Language](https://img.shields.io/badge/language-中文-blue.svg)](./README.md)
[![Language](https://img.shields.io/badge/language-English-green.svg)](./README_EN.md)

**Quick Navigation**: [Chinese Version](./README.md)

---

## Introduction

This repository provides complete Hackintosh OpenCore configuration files for **Dell Inspiron 15 Gaming 7567**, supporting **Windows 11 + macOS 13 Ventura + macOS 15 Sequoia + macOS 26 Tahoe** quad-boot setup.

| Metadata | Value |
|----------|-------|
| **OpenCore Version** | 1.0.7-RELEASE |
| **Core Framework** | OpCore-Simplify / OCLP Dynamic Patch Support |
| **Compatible Model** | Dell Inspiron 15 Gaming 7567 (Kaby Lake 7th Gen) |

---

## Hardware Specifications

| Component | Model | macOS Support |
|-----------|-------|---------------|
| **CPU** | Intel Core i5-7300HQ / i7-7700HQ | ✅ Native |
| **iGPU** | Intel HD Graphics 630 | ✅ Supported, requires OCLP patch |
| **dGPU** | NVIDIA GTX 1050/1050Ti | ❌ Disabled (-wegnoegpu) |
| **Audio** | Realtek ALC256 | ✅ Native on macOS 13/15, USB audio on macOS 26 |
| **WLAN** | Intel Wireless | ✅ itlwm + HeliPort |
| **BT** | Intel Bluetooth | ✅ IntelBluetoothFirmware + BlueToolFixup |
| **Trackpad** | I2C + PS2 Hybrid | ✅ VoodooI2C |
| **Eth** | Realtek RTL8111 | ✅ RealtekRTL8111.kext |

---

## Directory Structure

```
├── EFI_13/                          # macOS 13 Ventura专用 EFI
│   └── EFI/
│       ├── BOOT/                    # Universal BOOT loader
│       └── OC/
│           ├── ACPI/                # 5 SSDT patches
│           ├── Drivers/             # 4 UEFI drivers
│           ├── Kexts/               # 19 kernel extensions
│           ├── Resources/           # Graphics resources
│           ├── Tools/               # OpenCore tools
│           └── config.plist         # Main configuration
├── EFI_15/                          # macOS 15 Sequoia 专用 EFI
│   └── EFI/
│       ├── BOOT/
│       └── OC/
│           ├── ACPI/                # 6 SSDT (plus SSDT-XOSI)
│           ├── Drivers/             # 6 UEFI drivers
│           ├── Kexts/               # 27 kernel extensions
│           ├── Resources/
│           ├── Tools/
│           └── config.plist         # Main configuration
├── EFI_15_base_13/                  # Cross-version通用配置
│   └── OC/
│       ├── ACPI/
│       ├── Kexts/                   # 24 kernel extensions
│       └── config.plist             # Main configuration
├── docs/                            # Technical documentation
└── README_EN.md                     # This file (English)
```

---

## EFI Version Selection

| Config | Purpose | OpenCore Version | Kext Count | USB Mapping | WiFi Driver |
|--------|---------|-----------------|------------|-------------|-------------|
| `EFI_13/` | macOS 13专用 | 0.9.3 era | 19 | USBMap.kext | AirportItlwm |
| `EFI_15/` | macOS 15专用 | 1.0.x era | 27 | USBToolBox+UTBMap | itlwm |
| `EFI_15_base_13/` | Cross-version | 1.0.x era | 24 | USBToolBox+UTBMap | itlwm |

### Recommended Setup

| System Purpose | Recommended EFI |
|----------------|-----------------|
| macOS 13 (Ventura) - Stable Production | EFI_13/ |
| macOS 15 (Sequoia) - Main Recommended | EFI_15/ |
| macOS 26 (Tahoe) - Bleeding Edge | EFI_15/ + OCLP |
| Multi-boot Unified Loader | EFI_15_base_13/ |

---

## Quick Start

### 1. BIOS Settings

```
- Disable Legacy Option ROMs
- Change SATA operation to AHCI
- Disable Secure Boot
- Disable SGX
```

### 2. Create USB Installer

- Requires 16GB or larger USB drive
- See [16GB USB Stick Hackintosh Complete Installation Guide](./16GB-U%20盘黑苹果完整安装指南.md)

### 3. Install System

1. Boot OpenCore from USB
2. Select `Install macOS xxx`
3. Format target partition as APFS in Disk Utility
4. Start installation

### 4. Post-Installation

1. Copy corresponding EFI folder to ESP partition
2. Configure boot entries using OCAuxiliaryTools
3. **Reset NVRAM** to apply configuration

---

## Key Configuration Parameters

### NVRAM boot-args

```
keepsyms=1 debug=0x100 -v -igfxblt -vi2c-force-polling revpatch=sbvmm -wegnoegpu ipc_control_port_options=0 -amfipassbeta -ibtcompatbeta
```

### SIP Configuration

```
csr-active-config: 03080000 (Fully disable SIP, OCLP prerequisite)
```

---

## Important Notes

⚠️ **When modifying config.plist**:
- **DO NOT manually edit XML** - Use [OCAuxiliaryTools](https://github.com/ic005k/OCAuxiliaryTools) for visual editing
- Run `ocvalidate` after modifications
- **Reset NVRAM** after changing boot-args

⚠️ **Trackpad Configuration**:
- Disable `VoodooPS2Mouse.kext` and `VoodooPS2Trackpad.kext` to avoid I2C conflicts

⚠️ **dGPU Disabling**:
- Must use `-wegnoegpu` parameter, otherwise kernel panic will occur

---

## Documentation

| Document | Description |
|----------|-------------|
| [dell-7567-opencore-all-version-guide.md](./dell-7567-opencore-all-version-guide.md) | Full version deep configuration guide |
| [docs/config.plist-comparison.md](./docs/config.plist-comparison.md) | config.plist in-depth comparison report |
| [16GB USB Stick Installation Guide](./16GB-U%20盘黑苹果完整安装指南.md) | USB installer creation guide |

---

## References

- [Dortania OpenCore Official Guide](https://dortania.github.io/OpenCore-Install-Guide/)
- [OpenCore Legacy Patcher](https://github.com/dortania/OpenCore-Legacy-Patcher)
- [OCAuxiliaryTools](https://github.com/ic005k/OCAuxiliaryTools)
- [USBToolBox](https://github.com/USBToolBox/toolbox)
- [itlwm / HeliPort](https://github.com/intel-OSX/intel-OSX)

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Black screen on boot | Reset NVRAM |
| macOS boot entry not found | UEFI → ProtocolOverrides → FirmwareVolume = true |
| Trackpad unresponsive | Disable VoodooPS2Mouse/Trackpad |
| Slow performance, no acceleration | Run OCLP Post-Install Root Patch |
| Wi-Fi not working | Use itlwm.kext + HeliPort |
| dGPU consuming resources | Ensure -wegnoegpu in boot-args |

---

## License

MIT License.

---

**Quick Navigation**: [Chinese Version](./README.md)
