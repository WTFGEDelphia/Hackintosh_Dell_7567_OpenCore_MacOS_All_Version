# Hackintosh Dell 7567 OpenCore

[![Language](https://img.shields.io/badge/language-中文-blue.svg)](./README.md)
[![Language](https://img.shields.io/badge/language-English-green.svg)](./README_EN.md)

**快速跳转**: [English Version](./README_EN.md)

---

## 项目简介

本仓库为 **Dell Inspiron 15 Gaming 7567** 提供完整的 Hackintosh OpenCore 配置文件，支持 **Windows 11 + macOS 13 Ventura + macOS 15 Sequoia + macOS 26 Tahoe** 四系统共存方案。

| 元数据 | 内容 |
|--------|------|
| **OpenCore 版本** | 1.0.7-RELEASE |
| **核心框架** | OpCore-Simplify / OCLP 动态补丁支持 |
| **适用机型** | Dell Inspiron 15 Gaming 7567 (Kaby Lake 第 7 代) |

---

## 硬件规格

| 组件 | 型号 | macOS 支持状态 |
|------|------|---------------|
| **CPU** | Intel Core i5-7300HQ / i7-7700HQ | ✅ 原生支持 |
| **iGPU** | Intel HD Graphics 630 | ✅ 支持，需 OCLP 补丁 |
| **dGPU** | NVIDIA GTX 1050/1050Ti | ❌ 已禁用 (-wegnoegpu) |
| **Audio** | Realtek ALC256 | ✅ macOS 13/15 原生，macOS 26 需 USB 声卡 |
| **WLAN** | Intel Wireless | ✅ itlwm + HeliPort |
| **BT** | Intel Bluetooth | ✅ IntelBluetoothFirmware + BlueToolFixup |
| **Trackpad** | I2C + PS2 混合 | ✅ VoodooI2C |
| **Eth** | Realtek RTL8111 | ✅ RealtekRTL8111.kext |

---

## 目录结构

```
├── EFI_13/                          # macOS 13 Ventura 专用 EFI
│   └── EFI/
│       ├── BOOT/                    # 通用 BOOT 引导
│       └── OC/
│           ├── ACPI/                # 5 个 SSDT 补丁
│           ├── Drivers/             # 4 个 UEFI 驱动
│           ├── Kexts/               # 19 个内核扩展
│           ├── Resources/           # 图形资源
│           ├── Tools/               # OpenCore 工具
│           └── config.plist         # 主配置
├── EFI_15/                          # macOS 15 Sequoia 专用 EFI
│   └── EFI/
│       ├── BOOT/
│       └── OC/
│           ├── ACPI/                # 6 个 SSDT (多 SSDT-XOSI)
│           ├── Drivers/             # 6 个 UEFI 驱动
│           ├── Kexts/               # 27 个内核扩展
│           ├── Resources/
│           ├── Tools/
│           └── config.plist         # 主配置
├── EFI_15_base_13/                  # 跨版本通用配置
│   └── OC/
│       ├── ACPI/
│       ├── Kexts/                   # 24 个内核扩展
│       └── config.plist             # 主配置
├── docs/                            # 技术文档
└── README.md                        # 本文档 (中文)
```

---

## EFI 版本选择

| 配置 | 定位 | OpenCore 版本 | Kext 数量 | USB 映射 | WiFi 驱动 |
|------|------|--------------|-----------|----------|-----------|
| `EFI_13/` | macOS 13 专用 | 0.9.3 时期 | 19 | USBMap.kext | AirportItlwm |
| `EFI_15/` | macOS 15 专用 | 1.0.x 时期 | 27 | USBToolBox+UTBMap | itlwm |
| `EFI_15_base_13/` | 跨版本通用 | 1.0.x 时期 | 24 | USBToolBox+UTBMap | itlwm |

### 推荐方案

| 系统定位 | 推荐 EFI |
|----------|----------|
| macOS 13 (Ventura) - 稳定生产 | EFI_13/ |
| macOS 15 (Sequoia) - 主力推荐 | EFI_15/ |
| macOS 26 (Tahoe) - 尝鲜环境 | EFI_15/ + OCLP |
| 多系统统一引导 | EFI_15_base_13/ |

---

## 快速开始

### 1. BIOS 设置

```
- Disable Legacy Option ROMs
- Change SATA operation to AHCI
- Disable Secure Boot
- Disable SGX
```

### 2. 制作 USB 安装器

- 需要 16GB 或以上 U 盘
- 参考 [16GB-U 盘黑苹果完整安装指南](./16GB-U%20盘黑苹果完整安装指南.md)

### 3. 安装系统

1. U 盘引导 OpenCore
2. 选择 `Install macOS xxx`
3. 磁盘工具格式化目标分区 (APFS)
4. 开始安装

### 4. 安装后配置

1. 将对应 EFI 文件夹复制到 ESP 分区
2. 使用 OCAuxiliaryTools 配置启动项
3. **重置 NVRAM** 使配置生效

---

## 关键配置参数

### NVRAM boot-args

```
keepsyms=1 debug=0x100 -v -igfxblt -vi2c-force-polling revpatch=sbvmm -wegnoegpu ipc_control_port_options=0 -amfipassbeta -ibtcompatbeta
```

### SIP 配置

```
csr-active-config: 03080000 (完全关闭 SIP，OCLP 前置需求)
```

---

## 重要注意事项

⚠️ **修改 config.plist 时**:
- **严禁手动编辑 XML** - 使用 [OCAuxiliaryTools](https://github.com/ic005k/OCAuxiliaryTools) 可视化编辑
- 修改后运行 `ocvalidate` 验证
- 修改 boot-args 后必须执行 **Reset NVRAM**

⚠️ **触控板配置**:
- 禁用 `VoodooPS2Mouse.kext` 和 `VoodooPS2Trackpad.kext` 避免 I2C 冲突

⚠️ **独显屏蔽**:
- 必须使用 `-wegnoegpu` 参数，否则会导致内核恐慌

---

## 文档资源

| 文档 | 说明 |
|------|------|
| [dell-7567-opencore-all-version-guide.md](./docs/dell-7567-opencore-all-version-guide.md) | 全版本深度配置指南 |
| [docs/config.plist-comparison.md](./docs/config.plist-comparison.md) | config.plist 深度对比报告 |
| [16GB-U 盘黑苹果完整安装指南.md](./docs/16GB-U%20盘黑苹果完整安装指南.md) | USB 安装器制作指南 |

---

## 参考资源

- [Dortania OpenCore 官方文档](https://dortania.github.io/OpenCore-Install-Guide/)
- [OpenCore 下载](https://github.com/acidanthera/OpenCorePkg/releases)
- [Munki 脚本](https://github.com/munki/macadmin-scripts)
- [MountEFI](https://github.com/corpnewt/MountEFI)
- [ProperTree](https://github.com/corpnewt/ProperTree)
- [OCAuxiliaryTools](https://github.com/ic005k/OCAuxiliaryTools)
- [USBToolBox](https://github.com/USBToolBox/toolbox)
- [itlwm / HeliPort](https://github.com/intel-OSX/intel-OSX)
- [OpenCore Legacy Patcher](https://github.com/dortania/OpenCore-Legacy-Patcher)
- [Apple 官方支持](https://support.apple.com/en-us/HT201372)

---

## 故障排查

| 问题 | 解决方案 |
|------|----------|
| 启动黑屏 | 执行 Reset NVRAM |
| 找不到 macOS 启动盘 | UEFI → ProtocolOverrides → FirmwareVolume = true |
| 触控板失灵 | 禁用 VoodooPS2Mouse/Trackpad |
| 系统卡顿无加速 | 运行 OCLP Post-Install Root Patch |
| Wi-Fi 无法使用 | 改用 itlwm.kext + HeliPort |
| 独显占用资源 | 确保 boot-args 包含 -wegnoegpu |

---

## 许可

本项目采用 MIT 许可。

---

**快速跳转**: [English Version](./README_EN.md)
