# Dell 7567 (Kaby Lake) OpenCore 全版本深度配置指南

> **(Win11 + macOS 13 Ventura + macOS 15 Sequoia + macOS 26 Tahoe 三/四系统共存方案)**

| 元数据                  | 内容                                                 |
| ----------------------- | ---------------------------------------------------- |
| **文档版本**      | 2026-04-03 v2.0 (整合版)                             |
| **OpenCore 版本** | 1.0.7-RELEASE                                        |
| **核心框架**      | OpCore-Simplify / OCLP 动态补丁支持                  |
| **适用机型**      | Dell Inspiron 15 Gaming 7567 (Kaby Lake 第 7 代酷睿) |
| **参考配置**      | EFI_13 / EFI_15 / EFI_15_base_13                     |

---

## ⚠️ 前言：重要警告

在开始之前，请务必阅读并理解以下风险：

### 风险告知

| 风险项                 | 说明                                   | 预防措施                                      |
| ---------------------- | -------------------------------------- | --------------------------------------------- |
| **数据丢失**     | 操作分区和引导加载程序可能导致数据丢失 | ⚠️ 操作前务必备份所有重要数据               |
| **系统无法启动** | 错误的配置可能导致任一系统无法启动     | ⚠️ 准备好 U 盘引导介质作为备用              |
| **NVRAM 风险**   | 错误配置可能导致启动循环               | ⚠️ 每次修改 config.plist 后执行 Reset NVRAM |
| **电源风险**     | 操作过程中断电可能导致分区损坏         | ⚠️ 确保笔记本连接电源适配器                 |

### 技术 prerequisite

**本指南假设您已具备以下知识：**

- OpenCore 目录结构基础认知
- Kext/ACPI 基本概念
- config.plist 编辑能力
- UEFI 启动项管理

**零基础用户请先学习：**

1. [Dortania&#39;s OpenCore Install Guide](https://dortania.github.io/OpenCore-Install-Guide/)
2. [OpenCore Post-Install Guide](https://dortania.github.io/OpenCore-Post-Install/)

---

## 第一章：硬件规格与兼容性矩阵

### 1.1 Dell 7567 硬件清单

| 组件               | 型号/规格                        | macOS 兼容性  | 驱动策略                                               |
| ------------------ | -------------------------------- | ------------- | ------------------------------------------------------ |
| **CPU**      | Intel Core i5-7300HQ / i7-7700HQ | ✅ 原生支持   | `SSDT-PLUG` 启用 XCPM 电源管理                       |
| **iGPU**     | Intel HD Graphics 630            | ✅ 支持       | `WhateverGreen.kext` + OCLP Root Patch (macOS 15/26) |
| **dGPU**     | NVIDIA GeForce GTX 1050/1050Ti   | ❌ 不支持     | 必须通过 `-wegnoegpu` 彻底屏蔽                       |
| **Audio**    | Realtek ALC256                   | ⚠️ 版本差异 | macOS 13/15 原生; macOS 26 需 USB 声卡或 OCLP          |
| **WLAN**     | Intel Wireless 3160              | ✅ 支持       | `itlwm.kext` + `HeliPort` (规避多系统 KP 风险)     |
| **BT**       | Intel Bluetooth                  | ✅ 支持       | `IntelBluetoothFirmware.kext` + `-ibtcompatbeta`   |
| **Trackpad** | I2C / PS2 混合总线               | ✅ 支持       | `VoodooI2C` + 屏蔽 PS2 鼠标驱动                      |
| **Eth**      | Realtek RTL8111                  | ✅ 支持       | `RealtekRTL8111.kext`                                |
| **SATA**     | HM170/QM170 AHCI                 | ✅ 支持       | 原生支持                                               |

### 1.2 硬件注意事项

#### 独显屏蔽

GTX 1050/1050Ti 在 macOS 下无驱动支持，必须通过以下参数禁用：

```
boot-args: -wegnoegpu
```

否则会导致内核恐慌 (Kernel Panic)。

#### 触控板配置

Dell 7567 的触控板采用 I2C + PS2 混合总线设计。同时开启 PS2 鼠标/触控板驱动会导致：

- 中断抢占
- 系统严重卡顿
- 多指手势失效

**解决方案**：禁用 `VoodooPS2Mouse.kext` 和 `VoodooPS2Trackpad.kext`。

#### 音频架构变更

macOS 26 (Tahoe) 重构了音频协议栈，原生移除了 `AppleHDA.kext`：

- macOS 13/15: `AppleALC.kext` 原生支持 ALC256
- macOS 26: 建议使用 USB 声卡或运行 OCLP Root Patch 恢复

---

## 第二章：三系统共存架构设计

### 2.1 存储布局方案

```
┌─────────────────────────────────────────────────────────┐
│                    物理 SSD (示例：512GB)                │
├─────────────────────────────────────────────────────────┤
│  [ESP] EFI System Partition (≥200MB)                    │
│   └─ OpenCore 全局引导                                  │
│     ├─ EFI/BOOT/BOOTx64.efi                             │
│     └─ EFI/OC/                                          │
│         ├─ config.plist (统一配置)                      │
│         ├─ Kexts/ (24 个驱动)                           │
│         ├─ ACPI/ (6 个 SSDT)                            │
│         ├─ Drivers/ (6 个 UEFI 驱动)                     │
│         └─ Resources/ (图形资源)                        │
├─────────────────────────────────────────────────────────┤
│  [NTFS] Windows 11 系统分区                             │
├─────────────────────────────────────────────────────────┤
│  [APFS Container]                                       │
│   ├─ Mac 13 (macOS 13 Ventura 卷)                       │
│   ├─ Mac 15 (macOS 15 Sequoia 卷 - 主力推荐)            │
│   └─ Mac 26 (macOS 26 Tahoe 卷 - 尝鲜可选)              │
└─────────────────────────────────────────────────────────┘
```

### 2.2 分区详细步骤

#### 步骤 1: 创建唯一 ESP 分区

- 大小：≥200MB
- 格式：FAT32
- 用途：OpenCore 全局引导

#### 步骤 2: Windows 11 部署

1. 优先划分 NTFS 分区
2. 完成 Windows 11 安装
3. 禁用 Windows 快速启动 (Fast Startup)

#### 步骤 3: APFS Container 部署

1. 将剩余空间格式化为单一 APFS Container
2. 创建三个独立卷宗：
   - `Mac 13` - 稳定生产环境
   - `Mac 15` - 主力推荐系统
   - `Mac 26` - 研究尝鲜环境

**APFS 优势**：

- 底层数据完全隔离
- 动态共享空间
- 无需预先分配固定大小

### 2.3 引导流程

```
┌─────────────┐
│  开机按 F12  │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  BIOS 启动菜单│
│  选择 UEFI Boot│
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ OpenCore Picker│
│ 图形化选择系统  │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Install macOS │
│ 或已安装系统   │
└─────────────┘
```

---

## 第三章：EFI 版本选择指南

### 3.1 三个 EFI 配置对比

| 配置项                   | EFI_13         | EFI_15            | EFI_15_base_13    |
| ------------------------ | -------------- | ----------------- | ----------------- |
| **定位**           | macOS 13 专用  | macOS 15 专用     | 跨版本通用配置    |
| **OpenCore 版本**  | 0.9.3 时期     | 1.0.x 时期        | 1.0.x 时期        |
| **Kext 数量**      | 19 个          | 27 个             | 24 个             |
| **SSDT 数量**      | 5 个           | 6 个              | 6 个              |
| **UEFI 驱动**      | 4 个           | 6 个              | 6 个              |
| **SIP 状态**       | 启用 (0x00)    | 禁用 (0x0803)     | 禁用 (0x0803)     |
| **USB 映射**       | USBMap.kext    | USBToolBox+UTBMap | USBToolBox+UTBMap |
| **WiFi 驱动**      | AirportItlwm   | itlwm             | itlwm             |
| **SMBIOS**         | MacBookPro14,1 | MacBookPro14,3    | MacBookPro14,3    |
| **LauncherOption** | Disabled       | Full              | Full              |

### 3.2 详细差异分析

#### ACPI 配置差异

| SSDT 文件               | EFI_13 | EFI_15 | EFI_15_base_13 | 说明          |
| ----------------------- | ------ | ------ | -------------- | ------------- |
| SSDT-PLUG.aml           | ✅     | ✅     | ✅             | CPU 电源管理  |
| SSDT-PNLF.aml           | ✅     | ✅     | ✅             | 亮度控制      |
| SSDT-EC-USBX-LAPTOP.aml | ✅     | ✅     | ✅             | 嵌入式控制器  |
| SSDT-GPI0.aml           | ✅     | ✅     | ✅             | I2C 触控板    |
| SSDT-DXTR.aml           | ✅     | ✅     | ✅             | 深度睡眠      |
| **SSDT-XOSI.aml** | ❌     | ✅     | ✅             | _OSI 方法替换 |

#### ACPI Patch 差异

| Patch 名称                    | EFI_13 | EFI_15 | EFI_15_base_13 |
| ----------------------------- | ------ | ------ | -------------- |
| BRT6 to XRT6                  | ✅     | ✅     | ✅             |
| Gprw to Xprw                  | ✅     | ✅     | ✅             |
| **PNLF to XNLF Rename** | ❌     | ✅     | ✅             |
| **_OSI to XOSI rename** | ❌     | ✅     | ✅             |

#### Kext 关键差异

| Kext 名称                        | EFI_13     | EFI_15      | EFI_15_base_13 | 说明           |
| -------------------------------- | ---------- | ----------- | -------------- | -------------- |
| **USBMap.kext**            | ✅ Enabled | ❌ 移除     | ❌ 移除        | 旧版静态方案   |
| **USBToolBox.kext**        | ❌         | ✅          | ✅             | 新版引擎方案   |
| **UTBMap.kext**            | ❌         | ✅          | ✅             | USB 映射配置   |
| **AirportItlwm.kext**      | ✅ Enabled | ❌ 移除     | ❌ 移除        | 版本绑定风险   |
| **itlwm.kext**             | ❌         | ✅          | ✅             | 通用 WiFi 驱动 |
| **AMFIPass.kext**          | ❌         | ✅          | ✅             | AMFI 绕过      |
| **RestrictEvents.kext**    | ❌         | ✅          | ✅             | 机型欺骗       |
| **NVMeFix.kext**           | ❌         | ✅          | ✅             | NVMe 电源管理  |
| **CPUFriend.kext**         | ❌         | ✅          | ✅             | CPU 优化       |
| **IntelBTPatcher.kext**    | ❌         | ✅          | ✅             | 蓝牙补丁       |
| **VoodooPS2Mouse.kext**    | ✅         | ❌ Disabled | ❌ Disabled    | 触控板冲突     |
| **VoodooPS2Trackpad.kext** | ✅         | ❌ Disabled | ❌ Disabled    | I2C 冲突       |

#### Kext 版本对照表

| Kext                   | EFI_13 | EFI_15 | EFI_15_base_13 |
| ---------------------- | ------ | ------ | -------------- |
| Lilu                   | 1.6.6  | 1.7.3  | 1.7.3          |
| VirtualSMC             | 1.3.2  | 1.3.8  | 1.3.8          |
| WhateverGreen          | 1.6.5  | 1.7.1  | 1.7.1          |
| AppleALC               | 1.8.3  | 1.9.8  | 1.9.8          |
| VoodooI2C              | 2.8    | 2.9.1  | 2.9.1          |
| VoodooPS2              | 2.3.5  | 2.3.8  | 2.3.8          |
| RealtekRTL8111         | 2.4.2  | 3.0.4  | 3.0.4          |
| BlueToolFixup          | 2.6.7  | 2.7.2  | 2.7.2          |
| IntelBluetoothFirmware | 2.2.0  | 2.5.0  | 2.5.0          |

### 3.3 推荐配置决策树

```
是否需要 macOS 26?
├─ 是 → EFI_15_base_13 + USB 声卡
│
└─ 否 → 主要使用哪个系统？
    ├─ 仅 macOS 13 → EFI_13 (简洁)
    ├─ 仅 macOS 15 → EFI_15 (完整)
    └─ 双系统共存 → EFI_15_base_13 (通用)
```

### 3.4 最终推荐

**对于三系统共存用户，强烈推荐 `EFI_15_base_13` 配置：**

1. ✅ 完整继承 EFI_15 的核心功能
2. ✅ 支持 macOS 13/15/26 跨版本引导
3. ✅ 采用现代化 USBToolBox 方案
4. ✅ 使用通用 itlwm 驱动避免版本绑定
5. ✅ 精简了 3 个非必需 Kext

### 3.5 参考 EFI 仓库

#### 参考仓库：mishailovic/Hackintosh-Dell-7567-OpenCore_Ventura

**仓库地址**: [mishailovic/Hackintosh-Dell-7567-OpenCore_Ventura](https://github.com/mishailovic/Hackintosh-Dell-7567-OpenCore_Ventura)

**关键信息**:

| 项目                    | 内容                         |
| ----------------------- | ---------------------------- |
| **OpenCore 版本** | 0.9.3                        |
| **适用机型**      | Dell Inspiron 15 Gaming 7567 |
| **状态**          | 已归档 (ARCHIVED)            |
| **推荐 fork**     | SirDank 的更新版本           |

**功能支持状态**:

| 硬件   | 状态 | 说明                           |
| ------ | ---- | ------------------------------ |
| CPU    | ✅   | i5-7300HQ / i7-7700HQ 原生支持 |
| iGPU   | ✅   | Intel HD 630 正常工作          |
| dGPU   | ❌   | GTX 1050/1050Ti 已禁用         |
| Audio  | ⚠️ | 需要 ComboJack 修复            |
| WLAN   | ✅   | Intel Wireless，偶尔有 bug     |
| 读卡器 | ❌   | 不支持                         |
| HDMI   | ❌   | 不支持                         |

**BIOS 设置要求**:

```
- Disable Legacy Option ROMs
- Change SATA operation to AHCI
- Disable Secure Boot
- Disable SGX
```

**后安装修复**:

| 问题        | 解决方案                                                |
| ----------- | ------------------------------------------------------- |
| USB Mapping | 需要匹配 SMBIOS 模型重新映射                            |
| 平滑滚动    | 使用[MOS](https://mos.caldis.me/)                          |
| 音频输出    | 安装[ComboJack](https://github.com/HackintoshDR/ComboJack) |
| 睡眠修复    | 参考 Dortania Post-Install 指南                         |

---

## 第四章：U 盘安装器制作指南

### 4.1 硬件要求

| 项目                 | 要求         | 说明                |
| -------------------- | ------------ | ------------------- |
| **U 盘**       | 32GB 或更大  | 建议 USB 3.0 及以上 |
| **另一台 Mac** | macOS 10.13+ | 用于制作启动盘      |
| **网络连接**   | 稳定网络     | 下载 macOS 安装器   |

### 4.2 U 盘分区结构

```
┌─────────────────────────────────────────────┐
│              32GB U 盘                       │
├─────────────────────────────────────────────┤
│  [主分区] MyVolume (HFS+, 30-31GB)          │
│   ├─ macOS 安装器文件                        │
│   └─ 系统安装数据                           │
├─────────────────────────────────────────────┤
│  [EFI 分区] FAT32 (200MB)                   │
│   ├─ EFI/BOOT/BOOTx64.efi                   │
│   └─ EFI/OC/                                │
│       ├─ OpenCore.efi                       │
│       ├─ config.plist                       │
│       ├─ Kexts/                             │
│       ├─ ACPI/                              │
│       ├─ Drivers/                           │
│       └─ Resources/                         │
└─────────────────────────────────────────────┘
```

### 4.3 完整制作流程

#### 步骤 1: 下载 macOS 安装器

**方法 A: 使用 installinstallmacos.py 脚本（推荐）**

```bash
# 创建工作目录
mkdir -p ~/macOS-installer && cd ~/macOS-installer

# 下载脚本
curl https://raw.githubusercontent.com/munki/macadmin-scripts/main/installinstallmacos.py > installinstallmacos.py

# 运行脚本（会列出所有可用的 macOS 版本）
sudo python3 installinstallmacos.py
```

**脚本会显示类似以下的版本列表：**

```
#   ProductID    Version    Build   Post Date  Title
1   061-26578    10.14.5   18F2059  2019-10-14  macOS Mojave
...
16  093-22004    15.7.2    24G325   2025-11-03  macOS Sequoia
17  122-10756    26.4      25E246   2026-03-24  macOS Tahoe
...
```

**选择版本**: 输入对应数字（如 `17` 下载 macOS Tahoe 26.4），等待下载完成（约 15-30 分钟，取决于网络速度）

**方法 B: 直接下载 InstallAssistant.pkg**

从 Apple 官方渠道或 [Dortania Builds](https://dortania.github.io/OpenCore-Post-Install/universalOS.html) 下载对应版本的 InstallAssistant.pkg，双击安装后会在 `/Applications` 生成 `Install macOS xxx.app`。

**下载后的文件位置：**

- 使用脚本：`~/macOS-installer/Install_macOS_xxx.dmg`
- 直接下载：`/Applications/Install macOS xxx.app`

#### 步骤 2: 格式化 U 盘

```bash
# 查看磁盘列表
diskutil list

# 格式化 U 盘 (假设 /dev/disk2)
diskutil eraseDisk JHFS+ MyVolume GPTFormat /dev/disk2
```

#### 步骤 3: 创建可启动 U 盘

```bash
sudo /Applications/Install\ macOS\ xxx.app/Contents/Resources/createinstallmedia \
  --volume /Volumes/MyVolume \
  --nointeraction
```

#### 步骤 4: 写入 OpenCore 引导

**重要提示**: 推荐使用 **EFI_15_base_13** 作为 U 盘的 EFI 配置，它具有最佳的跨版本兼容性。

```bash
# 挂载 U 盘 EFI 分区
diskutil mount disk2s1

# 方法 A: 复制 EFI_15_base_13 配置（推荐 - 终极兼容）
# 假设 EFI_15_base_13 位于 ~/Downloads/EFI_15_base_13
cp -R ~/Downloads/EFI_15_base_13/OC/* /Volumes/EFI/EFI/OC/

# 方法 B: 复制标准 OpenCore 文件
# cp -R ~/OpenCore-1.0.7/EFI/BOOT /Volumes/EFI/
# cp -R ~/OpenCore-1.0.7/EFI/OC /Volumes/EFI/EFI/

# 创建 Resources 目录（如果不存在）
mkdir -p /Volumes/EFI/EFI/OC/Resources
```

**为什么选择 EFI_15_base_13？**

| 特性                     | EFI_15_base_13 优势                   |
| ------------------------ | ------------------------------------- |
| **跨版本兼容**     | 支持 macOS 13/15/26 统一引导          |
| **USBToolBox**     | 现代 USB 映射方案，系统升级容错性更好 |
| **itlwm WiFi**     | 通用驱动，避免版本绑定问题            |
| **NVRAM 清理**     | 完整的 Delete 配置，确保参数生效      |
| **FirmwareVolume** | 启用 Protocol，正确显示 macOS 安装盘  |

#### 步骤 5: 配置 config.plist

1. 使用 **OCAuxiliaryTools** 或 **ProperTree** 编辑
2. 关键配置检查：

   - `PlatformInfo -> Generic`: SMBIOS 设置
   - `NVRAM -> Add -> boot-args`: 启动参数
   - `Kernel -> Add`: Kexts 路径
   - `UEFI -> Drivers`: 驱动列表
3. 验证配置：

```bash
ocvalidate /Volumes/EFI/EFI/OC/config.plist
```

### 4.4 64GB U 盘多系统安装器共存方案

如果您想在同一个 U 盘上存放多个 macOS 安装器（如同时存放 macOS 15 和 macOS 26），可以使用以下方案：

#### 分区策略

```
┌─────────────────────────────────────────────────────────┐
│              64GB U 盘                                    │
├─────────────────────────────────────────────────────────┤
│  [分区 1] SequoiaInstaller (21GB, HFS+)                 │
│   └─ macOS 15 Sequoia 安装器                            │
├─────────────────────────────────────────────────────────┤
│  [分区 2] TahoeInstaller (21GB, HFS+)                   │
│   └─ macOS 26 Tahoe 安装器                              │
├─────────────────────────────────────────────────────────┤
│  [分区 3] Data (21GB, exFAT)                            │
│   └─ 通用数据存储（Windows/Mac 共用）                    │
└─────────────────────────────────────────────────────────┘
```

#### 操作步骤

**步骤 1: 使用磁盘工具分区**

1. 打开「磁盘工具」
2. 选择 U 盘根设备（不是分区）
3. 点击「分区」
4. 添加分区并设置：
   - **分区 1**: 名称 `SequoiaInstaller`, 格式 `Mac OS 扩展（日志式）`, 大小 `16GB`
   - **分区 2**: 名称 `TahoeInstaller`, 格式 `Mac OS 扩展（日志式）`, 大小 `16GB`
   - **分区 3**: 名称 `Data`, 格式 `exFAT`, 大小 `剩余空间`

**步骤 2: 分别写入安装器**

```bash
# 写入 macOS 15 Sequoia
sudo /Applications/Install\ macOS\ Sequoia.app/Contents/Resources/createinstallmedia \
  --volume /Volumes/SequoiaInstaller

# 写入 macOS 26 Tahoe
sudo /Applications/Install\ macOS\ Tahoe.app/Contents/Resources/createinstallmedia \
  --volume /Volumes/TahoeInstaller
```

**步骤 3: 添加 OpenCore 引导**

将 EFI 文件夹挂载到第一个分区的 EFI 分区中：

```bash
# 挂载 EFI 分区（通常在第一个分区）
diskutil list  # 找到 EFI 分区，如 disk2s1
sudo diskutil mount disk2s1

# 复制 EFI_15_base_13 配置
cp -R ~/Downloads/EFI_15_base_13/OC/* /Volumes/EFI/EFI/OC/
```

#### 如何使用

1. 将 U 盘插入电脑
2. 开机按 F12 进入启动菜单
3. 选择 `EFI Boot` 进入 OpenCore
4. 在 OpenCore 中选择对应的安装器

### 4.5 替代方案：OpCore-Simplify 工具

如果您不想手动制作 U 盘安装器，可以使用 **OpCore-Simplify** 工具自动生成 EFI。

**仓库地址**: [lzhoang2801/OpCore-Simplify](https://github.com/lzhoang2801/OpCore-Simplify)

#### 工具介绍

一个简化 OpenCore EFI 制作的工具，默认向下兼容，但需要根据机型自行适配。

#### 核心功能

| 功能                     | 说明                                            |
| ------------------------ | ----------------------------------------------- |
| **全面的硬件支持** | CPU: Nehalem → Arrow Lake (1-15 代)            |
| **GPU 支持**       | Intel iGPU (Iron Lake → Ice Lake), AMD, NVIDIA |
| **macOS 支持**     | High Sierra → macOS Tahoe                      |
| **ACPI 自动检测**  | 集成 SSDTTime，支持常见补丁                     |
| **自动更新**       | 自动从 Dortania 和 GitHub Releases 更新         |

#### 支持的 ACPI 补丁

| 补丁类型   | 说明                                        |
| ---------- | ------------------------------------------- |
| FakeEC     | 嵌入式控制器模拟                            |
| FixHPET    | HPET 修复                                   |
| PLUG       | CPU 电源管理                                |
| RTCAWAC    | RTC/AWAC 修复                               |
| 自定义补丁 | CPU 调度、UNC0 禁用、RTC 设备创建、睡眠修复 |

#### EFI 配置优化选项

- GPU ID 欺骗
- CpuTopologyRebuild (大小核 CPU)
- SIP 配置
- CPU ID 欺骗
- 不支持 SMBIOS 启动绕过
- NVRAM 条目配置
- ResizeAppleGpuBars
- iGPU 配置
- OCLP 配置
- 网络设备和存储控制器
- SMBIOS 优化
- 旧款 Intel CPU 电源管理
- itlwm WiFi 配置

#### 使用流程

```
1. 下载 OpCore-Simplify (ZIP 或 clone)
       ↓
2. 运行启动脚本 (Windows: .bat / macOS: .command / Linux: .py)
       ↓
3. 选择硬件报告 (推荐) 或手动加载 ACPI 表
       ↓
4. 选择 macOS 版本和自定义配置
       ↓
5. 构建 OpenCore EFI
       ↓
6. USB 映射
       ↓
7. 创建 USB 安装器并安装 macOS
```

#### 重要说明

1. **OCLP 支持**: 成功安装后，如果需要 OCLP，只需应用根补丁激活缺失功能
2. **AMD GPU**: 应用 OCLP 根补丁后，移除 `-radvesa`/`-amd_no_dgpu_accel` 启动参数

### 4.5 重要提示

| 事项                    | 说明                            |
| ----------------------- | ------------------------------- |
| **USB 端口映射**  | macOS 11.3+ 需要自定义 USB 映射 |
| **XhciPortLimit** | macOS 11.3+ 此限制已失效        |
| **U 盘名称**      | 必须为 `MyVolume`             |
| **16GB 限制**     | 8GB U 盘无法完成                |

---

## 第五章：config.plist 深度配置详解

### 5.1 ACPI 配置

#### SSDT 加载列表

| 顺序 | SSDT 名称               | 路径                         | 启用 | 说明                      |
| ---- | ----------------------- | ---------------------------- | ---- | ------------------------- |
| 1    | SSDT-XOSI.aml           | ACPI/SSDT-XOSI.aml           | ✅   | _OSI 方法替换 (macOS 15+) |
| 2    | SSDT-PLUG.aml           | ACPI/SSDT-PLUG.aml           | ✅   | CPU 电源管理              |
| 3    | SSDT-PNLF.aml           | ACPI/SSDT-PNLF.aml           | ✅   | 亮度控制                  |
| 4    | SSDT-EC-USBX-LAPTOP.aml | ACPI/SSDT-EC-USBX-LAPTOP.aml | ✅   | 嵌入式控制器              |
| 5    | SSDT-GPI0.aml           | ACPI/SSDT-GPI0.aml           | ✅   | I2C 触控板                |
| 6    | SSDT-DXTR.aml           | ACPI/SSDT-DXTR.aml           | ✅   | 深度睡眠                  |

#### ACPI Patch 配置

| Patch 名称   | 启用 | Find (Base64)    | Replace (Base64) | 说明             |
| ------------ | ---- | ---------------- | ---------------- | ---------------- |
| BRT6 to XRT6 | ✅   | `QlJUNgKgC5M=` | `WFJUNgKgC5M=` | 亮度键修复       |
| Gprw to Xprw | ✅   | `R1BSVwI=`     | `WFBSVwI=`     | 睡眠修复         |
| PNLF to XNLF | ✅   | `UE5MRg==`     | `WE5MRg==`     | 亮度 SSDT 重命名 |
| _OSI to XOSI | ✅   | `X09TSQ==`     | `X09TSQ==`     | Windows 检测绕过 |

#### ACPI Quirks 设置

| 设置项           | 推荐值    | 说明               |
| ---------------- | --------- | ------------------ |
| FadtEnableReset  | `false` | -                  |
| NormalizeHeaders | `false` | -                  |
| RebaseRegions    | `false` | -                  |
| ResetHwSig       | `false` | -                  |
| ResetLogoStatus  | `true`  | 启用后显示启动徽标 |
| SyncTableIds     | `false` | -                  |

### 5.2 Kernel Kexts 配置

#### 核心 Kext 加载顺序

```
1. Lilu.kext              (核心框架，必须优先)
2. VirtualSMC.kext        (SMC 仿真)
3. WhateverGreen.kext     (显卡补丁)
4. AppleALC.kext          (声卡驱动)
5. SMCBatteryManager.kext (电池管理)
6. SMCProcessor.kext      (CPU 监控)
7. SMCDellSensors.kext    (Dell 传感器)
8. VoodooI2C 系列         (触控板)
9. VoodooPS2Controller.kext (PS2 键盘)
10. RealtekRTL8111.kext   (有线网卡)
11. IntelBluetoothFirmware.kext (蓝牙固件)
12. BlueToolFixup.kext    (蓝牙补丁)
13. AMFIPass.kext         (AMFI 绕过)
14. RestrictEvents.kext   (事件限制)
15. NVMeFix.kext          (NVMe 电源管理)
16. IntelBTPatcher.kext   (蓝牙补丁)
17. CPUFriend.kext        (CPU 优化)
18. CPUFriendDataProvider.kext (CPU 数据)
19. USBToolBox.kext       (USB 引擎)
20. UTBMap.kext           (USB 配置)
21. itlwm.kext            (WiFi 驱动)
22. VerbStub.kext         (视频输出)
```

**注意**：USBToolBox.kext 必须在 UTBMap.kext 之前加载！

#### 禁用/启用 Kext

| Kext 名称              | 状态        | 原因               |
| ---------------------- | ----------- | ------------------ |
| VoodooPS2Mouse.kext    | ❌ Disabled | 触控板冲突         |
| VoodooPS2Trackpad.kext | ❌ Disabled | I2C 冲突           |
| AirportItlwm.kext      | ❌ 移除     | 版本绑定风险       |
| USBMap.kext            | ❌ 移除     | 被 USBToolBox 替代 |

### 5.3 NVRAM 配置

#### boot-args 启动参数

```
keepsyms=1 debug=0x100 -v -igfxblt -vi2c-force-polling revpatch=sbvmm -wegnoegpu ipc_control_port_options=0 -amfipassbeta -ibtcompatbeta
```

| 参数                           | 作用                   | 必要性                     |
| ------------------------------ | ---------------------- | -------------------------- |
| `keepsyms=1`                 | 崩溃时显示符号信息     | 推荐                       |
| `debug=0x100`                | 启用调试日志           | 推荐                       |
| `-v`                         | 详细模式启动           | 调试用                     |
| `-igfxblt`                   | Intel 显卡亮度控制修复 | **必需**             |
| `-vi2c-force-polling`        | I2C 触控板强制轮询     | 推荐                       |
| `revpatch=sbvmm`             | OCLP 补丁策略          | **必需**             |
| `-wegnoegpu`                 | 禁用独显 GTX 1050      | **必需**             |
| `ipc_control_port_options=0` | 修复 Electron 应用崩溃 | 推荐                       |
| `-amfipassbeta`              | AMFIPass 跨版本生效    | **必需**             |
| `-ibtcompatbeta`             | Intel 蓝牙初始化修复   | **必需** (macOS 15+) |

#### csr-active-config (SIP)

```xml
<data>AwgAAA==</data>
<!-- Hex: 03 08 00 00 = 0x00000803 -->
```

**0x00000803 含义**：

- `CSR_VALID_APPLE_OS` (0x00000001)
- `CSR_ALLOW_UNTRUSTED_KEXTS` (0x00000002)
- `CSR_ALLOW_UNRESTRICTED_FS` (0x00000004)
- `CSR_ALLOW_TASK_FOR_PID` (0x00000008)
- `CSR_ALLOW_UNRESTRICTED_DARWIN_KEXTS` (0x00000800)

**注意**：完全关闭 SIP 是 OCLP Root Patch 的前置需求。

#### NVRAM Delete 配置

必须清理以下项目以防止旧参数覆盖：

```xml
<key>7C436110-AB2A-4BBB-A880-FE41995C9F82</key>
<array>
    <string>prev-lang:kbd</string>
    <string>boot-args</string>
    <string>csr-active-config</string>
    <string>SystemAudioVolume</string>
    <string>ForceDisplayRotationInEFI</string>
    <string>bluetoothExternalDongleFailed</string>
    <string>bluetoothInternalControllerInfo</string>
</array>
```

**重要性**：不清理 NVRAM 会导致修改不生效！

### 5.4 UEFI 驱动配置

#### 必需驱动列表

| 驱动                | 启用 | 说明                       |
| ------------------- | ---- | -------------------------- |
| OpenRuntime.efi     | ✅   | 内存管理，须与 OC 版本匹配 |
| OpenHfsPlus.efi     | ✅   | 读取 HFS+ 分区             |
| OpenCanopy.efi      | ✅   | 图形化引导界面             |
| AudioDxe.efi        | ✅   | 音频支持                   |
| ResetNvramEntry.efi | ✅   | NVRAM 清理工具             |
| apfs_aligned.efi    | ✅   | APFS 驱动 (macOS 15+)      |

#### APFS 配置

| 设置项           | 推荐值    | 说明               |
| ---------------- | --------- | ------------------ |
| EnableJumpstart  | `true`  | 动态抓取 APFS 驱动 |
| GlobalConnect    | `false` | -                  |
| HideVerbose      | `true`  | 隐藏详细日志       |
| JumpstartHotPlug | `false` | -                  |
| MinDate          | `-1`    | 无限制             |
| MinVersion       | `-1`    | 无限制             |

**注意**：启用 `EnableJumpstart` 后，`apfs_aligned.efi` 可选。

#### ProtocolOverrides 配置

| 设置项          | 推荐值    | 说明                               |
| --------------- | --------- | ---------------------------------- |
| FirmwareVolume  | `true`  | **必需** - 显示 macOS 安装盘 |
| AppleAudio      | `false` | -                                  |
| AppleBootPolicy | `false` | -                                  |

### 5.5 PlatformInfo 配置

#### SMBIOS 设置

| 设置项             | 推荐值                                   | 说明               |
| ------------------ | ---------------------------------------- | ------------------ |
| SystemProductName  | `MacBookPro14,3`                       | 更接近实际硬件     |
| MLB                | `C02712902QXJ1JH1M`                    | 需自行生成有效序列 |
| SystemSerialNumber | `C02TGBZ3HTD5`                         | 需自行生成有效序列 |
| SystemUUID         | `08ABDD0B-0876-4649-9C05-954A37DF098C` | 需自行生成         |
| ROM                | `oChAWYTF`                             | 匹配实际网卡的 MAC |
| ProcessorType      | `1537`                                 | Kaby Lake HQ 系列  |
| SpoofVendor        | `true`                                 | 启用厂商欺骗       |

**注意**：SMBIOS 信息需要通过 [Serial Generator](https://macserial.tech/) 自行生成有效的序列号。

### 5.6 Misc 配置

#### Boot 设置

| 设置项           | 推荐值                     | 说明                              |
| ---------------- | -------------------------- | --------------------------------- |
| LauncherOption   | `Full`                   | **推荐** - 写入 UEFI 启动项 |
| PickerMode       | `External`               | 图形化界面                        |
| PickerVariant    | `Acidanthera\GoldenGate` | 主题                              |
| PickerAttributes | `17`                     | 启用图形选项                      |
| Timeout          | `999`                    | 超时时间 (秒)                     |

**LauncherOption 对比**：

- `Disabled`: 不写入 NVRAM 启动项
- `Full`: **推荐** - 在 UEFI 中写入最高优先级引导项，抵抗 Windows 更新覆盖

---

## 第六章：USBToolBox 迁移指南

### 6.1 为什么要迁移？

| 方案                          | 架构        | 跨代兼容性 | 维护性     |
| ----------------------------- | ----------- | ---------- | ---------- |
| **USBMap.kext**         | 静态驱动    | ⚠️ 较差  | 需重新生成 |
| **USBToolBox + UTBMap** | 引擎 + 配置 | ✅ 优秀    | 易于更新   |

**USBToolBox 优势**：

1. 引擎分离设计，系统升级容错性更好
2. 支持 Windows 下自动生成映射
3. 对 macOS 14/15/26 的 USB 栈变更有更好的适应性

### 6.2 迁移步骤

请根据 [USBToolBox](https://github.com/USBToolBox/kext) 指南进行操作

#### 步骤 1: 准备新驱动

```bash
# 下载 USBToolBox 驱动
# 从 https://github.com/USBToolBox/kext/releases 下载

# 文件清单:
# - USBToolBox.kext (引擎)
# - UTBMap.kext (映射配置，需自定义生成)
```

#### 步骤 2: 更新 config.plist

1. 在 `Kernel -> Add` 中添加:

   ```xml
   <!-- USBToolBox.kext 必须在 UTBMap.kext 之前 -->
   <dict>
       <key>BundlePath</key>
       <string>USBToolBox.kext</string>
       <key>Enabled</key>
       <true/>
       ...
   </dict>
   <dict>
       <key>BundlePath</key>
       <string>UTBMap.kext</string>
       <key>Enabled</key>
       <true/>
       ...
   </dict>
   ```
2. 删除旧的 `USBMap.kext` 条目

#### 步骤 3: 删除旧文件

```bash
# 删除 USBMap.kext
rm -rf EFI/OC/Kexts/USBMap.kext
```

#### 步骤 4: 验证

```bash
# 验证配置
ocvalidate EFI/OC/config.plist

# 重启并执行 Reset NVRAM
```

### 6.3 验证清单

| 检查项       | 方法              | 预期结果         |
| ------------ | ----------------- | ---------------- |
| USB 3.0 速度 | 系统信息 → USB   | 显示 `5 Gb/s`  |
| 端口数量     | ioreg -l          | ≤ 15 个有效端口 |
| 所有端口可用 | 插入 USB 设备测试 | 正常识别         |

---

## 第七章：OCLP Root Patch 流程

### 7.1 为什么需要 OCLP？

macOS 15/26 对 Intel HD 630 不再提供原生图形加速，需要 OCLP 补丁：

| 功能            | 需要补丁 | 说明          |
| --------------- | -------- | ------------- |
| 图形加速        | ✅       | Metal 支持    |
| 核显驱动        | ✅       | Intel HD 630  |
| 音频 (macOS 26) | ⚠️     | AppleHDA 恢复 |

### 7.2 完整流程

```
1. 安装 macOS → 2. 联网 (HeliPort) → 3. 运行 OCLP → 4. Post-Install Root Patch → 5. 重启
```

#### 步骤 1: 连接网络

使用 **HeliPort** 应用连接 Wi-Fi：

```
itlwm.kext + HeliPort = 完美的 WiFi 解决方案
```

#### 步骤 2: 下载 OCLP

```bash
# 从 https://github.com/dortania/OpenCore-Legacy-Patcher/releases 下载
# 下载 OpenCore-Patcher.app
```

#### 步骤 3: 运行 Root Patch

1. 打开 **OpenCore-Legacy Patcher**
2. 点击 **Post-Install Root Patch**
3. 程序会自动扫描缺失的组件
4. 点击 **Start Patching**
5. 完成后重启

#### 步骤 4: 验证补丁

| 检查项   | 方法                 | 预期结果         |
| -------- | -------------------- | ---------------- |
| 图形加速 | 系统报告 → 显卡     | Metal 支持已启用 |
| 分辨率   | 系统偏好设置         | 可选择最高分辨率 |
| 音频     | 系统偏好设置 → 声音 | 输出设备可见     |

---

## 第八章：故障排查速查表

### 8.1 启动问题

| 问题现象        | 可能原因              | 解决方案                                           |
| --------------- | --------------------- | -------------------------------------------------- |
| 启动黑屏        | NVRAM 未重置          | 执行 Reset NVRAM                                   |
| 找不到 macOS 盘 | FirmwareVolume 未启用 | UEFI → ProtocolOverrides → FirmwareVolume = true |
| 卡在 Apple Logo | 缺少 -wegnoegpu       | boot-args 添加 -wegnoegpu                          |
| 无限重启        | ACPI 冲突             | 检查 SSDT 配置，尝试禁用 DXTR                      |

### 8.2 硬件问题

| 问题现象             | 可能原因          | 解决方案                             |
| -------------------- | ----------------- | ------------------------------------ |
| 触控板失灵           | PS2 驱动冲突      | 禁用 VoodooPS2Mouse/Trackpad         |
| 触控板卡顿           | 未启用 I2C 轮询   | boot-args 添加 -vi2c-force-polling   |
| 亮度无法调节         | PNLF 补丁问题     | 检查 SSDT-PNLF 和 PNLF to XNLF Patch |
| 无声音 (macOS 13/15) | layout-id 错误    | DeviceProperties → layout-id = 16   |
| 无声音 (macOS 26)    | AppleHDA 被移除   | 使用 USB 声卡或运行 OCLP             |
| Wi-Fi 无法使用       | AirportItlwm 冲突 | 改用 itlwm.kext + HeliPort           |
| 蓝牙无法使用         | 缺少补丁          | 添加 -ibtcompatbeta + IntelBTPatcher |

### 8.3 系统问题

| 问题现象     | 可能原因          | 解决方案                                     |
| ------------ | ----------------- | -------------------------------------------- |
| 系统卡顿     | 未运行 OCLP       | 执行 Post-Install Root Patch                 |
| VSCode 崩溃  | IPC 端口问题      | boot-args 确保 ipc_control_port_options=0    |
| 独显占用     | -wegnoegpu 未设置 | boot-args 添加 -wegnoegpu                    |
| SIP 无法关闭 | NVRAM 未清理      | Delete 中添加 boot-args 和 csr-active-config |

### 8.4 配置修改不生效

| 检查项               | 说明                                  |
| -------------------- | ------------------------------------- |
| 1. Reset NVRAM       | **最常用** - 每次修改后必须执行 |
| 2. NVRAM Delete      | 确保旧配置被清理                      |
| 3. 验证 config.plist | 使用 ocvalidate 验证                  |
| 4. 检查 EFI 挂载     | 确认修改的是正确分区的文件            |

---

## 附录 A: macOS 26 Tahoe 专属适配

### A.1 致命变更：AppleHDA 被移除

**苹果在 macOS 26 中彻底移除了 `AppleHDA.kext`。**

#### 解决方案对比

| 方案      | 适用场景      | 风险       | 音质     | 推荐度      |
| --------- | ------------- | ---------- | -------- | ----------- |
| USB 声卡  | 日常使用      | ⭐ 无风险  | ⭐⭐⭐⭐ | ✅ 强烈推荐 |
| OCLP-Mod  | 极度渴望内置  | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⚠️ 不推荐 |
| VoodooHDA | 不买 USB 声卡 | ⭐⭐       | ⭐⭐     | ⚠️ 备选   |

### A.2 Intel 蓝牙驱动需新参数

必须在 `boot-args` 中添加：

```
-ibtcompatbeta
```

### A.3 SMBIOS 支持

macOS 26 仅支持 2019 年后的有限机型，需要通过 `RestrictEvents.kext` + `revpatch=sbvmm` 进行 VMM 虚拟机伪装。

### A.4 官方建议

> "AppleHDA.kext is no longer present in macOS 26... use USB audio adapter for best compatibility."
>
> —— Dortania macOS 26 Tahoe 文档

**建议**：

- macOS 13: 稳定生产环境
- macOS 15: 主力推荐系统
- macOS 26: 研究尝鲜环境 (配合 USB 声卡)

---

## 附录 B: 系统版本选择建议

### B.1 三系统对比

| 维度                 | macOS 13         | macOS 15         | macOS 26           |
| -------------------- | ---------------- | ---------------- | ------------------ |
| **音频架构**   | ✅ 完整 AppleHDA | ✅ 完整 AppleHDA | ❌ AppleHDA 被移除 |
| **内置声卡**   | ✅ 完美工作      | ✅ 完美工作      | ❌ 需 USB 声卡     |
| **UI/功能**    | ⭐⭐⭐ 较旧      | ⭐⭐⭐⭐⭐ 现代  | ⭐⭐⭐⭐⭐ 最新    |
| **软件生态**   | ⭐⭐⭐           | ⭐⭐⭐⭐⭐       | ⭐⭐⭐⭐           |
| **驱动复杂度** | ⭐⭐ 简单        | ⭐⭐⭐ 中等      | ⭐⭐⭐⭐⭐ 激进    |
| **OCLP 需求**  | 仅需显卡         | 仅需显卡         | 显卡 + 音频        |

### B.2 推荐用途

| 系统               | 定位         | 推荐用途             |
| ------------------ | ------------ | -------------------- |
| **macOS 13** | 稳定生产环境 | 兼容性最佳，适合工作 |
| **macOS 15** | 主力推荐系统 | 完美平衡点           |
| **macOS 26** | 研究尝鲜环境 | 探索新特性           |

---

## 附录 C: 部署与后期维护工作流

### C.1 首次启动流程

```
1. 选择 Reset NVRAM → 自动重启
2. 再次进入 OpenCore，选择系统
3. 进入桌面后安装 HeliPort
4. 连接网络
5. 运行 OCLP Root Patch
6. 重启，享受完美体验
```

### C.2 日常维护

| 任务       | 频率   | 工具                |
| ---------- | ------ | ------------------- |
| 系统更新   | 按需   | 系统偏好设置        |
| Kext 更新  | 季度   | OCAT / ocvalidate   |
| NVRAM 清理 | 必要时 | ResetNvramEntry.efi |
| 备份 EFI   | 定期   | 手动复制            |

### C.3 配置验证命令

```bash
# 验证配置文件
ocvalidate EFI/OC/config.plist

# 检查 USB 端口
ioreg -l | grep -i "usb"

# 检查 Kext 加载
kextstat | grep -v com.apple

# 查看 SIP 状态
csrutil status
```

---

## 附录 D: 快速参考命令

### D.1 U 盘操作

```bash
# 查看磁盘列表
diskutil list

# 格式化 U 盘
diskutil eraseDisk JHFS+ MyVolume GPTFormat /dev/diskX

# 挂载 EFI 分区
diskutil mount diskXs1

# 弹出 U 盘
diskutil eject /Volumes/MyVolume
```

### D.2 macOS 安装器

```bash
# 下载 macOS 安装器
sudo ./installinstallmacos.py --workdir:~/macOSInstaller

# 创建可启动 U 盘
sudo /Applications/Install\ macOS\ xxx.app/Contents/Resources/createinstallmedia \
  --volume /Volumes/MyVolume \
  --nointeraction
```

### D.3 OpenCore 相关

```bash
# 验证 config.plist
ocvalidate /path/to/config.plist

# 下载 OpenCore
curl -L -o OpenCore.zip \
  https://github.com/acidanthera/OpenCorePkg/releases/download/X.X.X/OpenCore-X.X.X-RELEASE.zip
```

---

## 附录 E: EFI 参考资源与工具

### E.1 终极兼容 EFI: EFI_15_base_13

**推荐用于 U 盘制作和多系统共存环境**

| 项目                    | 内容                               |
| ----------------------- | ---------------------------------- |
| **定位**          | 跨版本通用配置（基于 EFI_13 合并） |
| **OpenCore 版本** | 1.0.x                              |
| **SIP 状态**      | 禁用 (0x0803) - OCLP 前置需求      |
| **USB 映射**      | USBToolBox + UTBMap                |
| **WiFi 驱动**     | itlwm                              |
| **SMBIOS**        | MacBookPro14,3                     |

**核心特性**:

| 特性                     | 说明                                       |
| ------------------------ | ------------------------------------------ |
| **Kext 数量**      | 24 个（精简版）                            |
| **SSDT 数量**      | 6 个（含 SSDT-XOSI）                       |
| **UEFI 驱动**      | 6 个（含 ResetNvramEntry, apfs_aligned）   |
| **LauncherOption** | Full - 写入 UEFI 启动项，抵抗 Windows 覆盖 |
| **FirmwareVolume** | true - 正确显示 macOS 安装盘               |

**Kext 列表**:

- Lilu 1.7.3, VirtualSMC 1.3.8, WhateverGreen 1.7.1, AppleALC 1.9.8
- VoodooI2C 2.9.1, VoodooPS2 2.3.8, RealtekRTL8111 3.0.4
- USBToolBox 1.2.0, itlwm 2.3.0
- AMFIPass, RestrictEvents, NVMeFix, CPUFriend
- IntelBluetoothFirmware 2.5.0, IntelBTPatcher, BlueToolFixup

**为什么推荐用于 U 盘？**

1. 跨版本兼容性最佳，可统一引导 macOS 13/15/26
2. USBToolBox 方案对系统升级有更好的容错性
3. itlwm 通用驱动避免版本绑定问题
4. 完整的 NVRAM Delete 配置确保参数生效

### E.2 Dell 7567 专用 EFI 参考

#### mishailovic/Hackintosh-Dell-7567-OpenCore_Ventura

| 项目                    | 内容                                                                                                                   |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| **仓库**          | [mishailovic/Hackintosh-Dell-7567-OpenCore_Ventura](https://github.com/mishailovic/Hackintosh-Dell-7567-OpenCore_Ventura) |
| **OpenCore 版本** | 0.9.3                                                                                                                  |
| **状态**          | 已归档 (ARCHIVED)                                                                                                      |
| **推荐 fork**     | SirDank 的更新版本                                                                                                     |

**功能支持**:

| 硬件   | 状态 | 说明                       |
| ------ | ---- | -------------------------- |
| CPU    | ✅   | i5-7300HQ / i7-7700HQ      |
| iGPU   | ✅   | Intel HD 630               |
| dGPU   | ❌   | GTX 1050/1050Ti 已禁用     |
| Audio  | ⚠️ | 需要 ComboJack 修复        |
| WLAN   | ✅   | Intel Wireless，偶尔有 bug |
| 读卡器 | ❌   | 不支持                     |
| HDMI   | ❌   | 不支持                     |

**BIOS 设置**:

```
- Disable Legacy Option ROMs
- Change SATA operation to AHCI
- Disable Secure Boot
- Disable SGX
```

### E.2 EFI 自动化工具

#### lzhoang2801/OpCore-Simplify

| 项目           | 内容                                                                       |
| -------------- | -------------------------------------------------------------------------- |
| **仓库** | [lzhoang2801/OpCore-Simplify](https://github.com/lzhoang2801/OpCore-Simplify) |
| **类型** | EFI 自动生成工具                                                           |
| **语言** | Python / Batch / Shell                                                     |

**支持范围**:

| 类别            | 支持范围                                        |
| --------------- | ----------------------------------------------- |
| **CPU**   | Nehalem/Westmere (1 代) → Arrow Lake (15 代)   |
| **iGPU**  | Iron Lake → Ice Lake                           |
| **dGPU**  | AMD, NVIDIA (Kepler/Pascal/Maxwell/Fermi/Tesla) |
| **macOS** | High Sierra → macOS Tahoe                      |

**核心功能**:

- ACPI 补丁自动检测 (SSDTTime 集成)
- Kext 自动更新 (Dortania + GitHub Releases)
- GPU ID 欺骗
- SIP 配置
- NVRAM 条目配置
- OCLP 配置支持
- itlwm WiFi 配置

### E.3 其他社区资源

| 资源                                                                          | 用途               |
| ----------------------------------------------------------------------------- | ------------------ |
| [Dortania OpenCore 官方文档](https://dortania.github.io/OpenCore-Install-Guide/) | 权威安装指南       |
| [OpenCore Legacy Patcher](https://github.com/dortania/OpenCore-Legacy-Patcher)   | 旧硬件补丁工具     |
| [OCAuxiliaryTools](https://github.com/ic005k/OCAuxiliaryTools)                   | 配置文件可视化工具 |
| [itlwm / HeliPort](https://github.com/intel-OSX/intel-OSX)                       | Intel 无线网卡驱动 |
| [USBToolBox](https://github.com/USBToolBox/toolbox)                              | USB 映射框架       |
| [MOS](https://mos.caldis.me/)                                                    | 平滑滚动修复       |
| [ComboJack](https://github.com/HackintoshDR/ComboJack)                           | 音频修复           |

---

## 附录 F: 修订历史

| 版本 | 日期       | 变更内容                |
| ---- | ---------- | ----------------------- |
| 1.0  | 2026-03-01 | 初始版本 (EFI_13)       |
| 1.5  | 2026-03-15 | 添加 EFI_15 支持        |
| 2.0  | 2026-04-03 | 全版本整合指南          |
| 2.1  | 2026-04-03 | 补充 EFI 参考资源与工具 |

---

**文档维护**: 本指南随 OpenCore 和 macOS 版本更新而持续优化
**最后更新**: 2026-04-03
**仓库**: [Hackintosh_Dell_7567_OpenCore_MacOS_All_Version](https://github.com/wangtf/Hackintosh_Dell_7567_OpenCore_MacOS_All_Version)
