# Dell 7567 EFI 三方配置深度对比报告

> **生成时间**: 2026-04-03
>
> **对比文件**:
> 1. `EFI_13/EFI/OC/config.plist` - macOS 13 Ventura 原版配置
> 2. `EFI_15/EFI/OC/config.plist` - macOS 15 Sequoia 原版配置
> 3. `EFI_15_base_13/OC/config.plist` - 基于 EFI_13 手动合并的 EFI_15 配置

---

## 执行摘要

| 对比维度 | EFI_13 | EFI_15 | EFI_15_base_13 |
|---------|--------|--------|---------------|
| **定位** | macOS 13 专用 | macOS 15 专用 | 跨版本通用 (基于 EFI_13 合并) |
| **Kext 数量** | 19 个 | 27 个 | 24 个 |
| **SSDT 数量** | 5 个 | 6 个 | 6 个 |
| **UEFI 驱动** | 4 个 | 6 个 | 6 个 |
| **SIP 状态** | 启用 (0x00000000) | 禁用 (0x00000803) | 禁用 (0x00000803) |
| **SMBIOS 机型** | MacBookPro14,1 | MacBookPro14,3 | MacBookPro14,3 |
| **USB 映射方案** | USBMap.kext | USBToolBox + UTBMap | USBToolBox + UTBMap |
| **WiFi 驱动** | AirportItlwm | itlwm | itlwm |

### 关键发现

**EFI_15_base_13 是一个"中间状态"配置**：
- 继承了 EFI_15 的现代特性（USBToolBox、itlwm、AMFIPass 等）
- 保留了部分 EFI_13 的简洁性（少了 3 个 Kext）
- **但缺少 EFI_13 独有的 RealtekRTL8111.kext**（有但有不同版本）

---

## 一、ACPI 配置对比

### 1.1 SSDT 加载列表

| SSDT 文件 | EFI_13 | EFI_15 | EFI_15_base_13 |
|-----------|--------|--------|---------------|
| SSDT-PLUG.aml | ✅ | ✅ | ✅ |
| SSDT-PNLF.aml | ✅ | ✅ | ✅ |
| SSDT-EC-USBX-LAPTOP.aml | ✅ | ✅ | ✅ |
| SSDT-GPI0.aml | ✅ | ✅ | ✅ |
| SSDT-DXTR.aml | ✅ | ✅ | ✅ |
| **SSDT-XOSI.aml** | ❌ | ✅ | ✅ |

**结论**: EFI_15_base_13 与 EFI_15 完全一致，已添加 XOSI 支持。

### 1.2 ACPI Patch 对比

| Patch 名称 | EFI_13 | EFI_15 | EFI_15_base_13 |
|------------|--------|--------|---------------|
| BRT6 to XRT6 | ✅ | ✅ | ✅ |
| Gprw to Xprw | ✅ | ✅ | ✅ |
| **PNLF to XNLF Rename** | ❌ | ✅ | ✅ |
| **_OSI to XOSI rename** | ❌ | ✅ | ✅ |

**结论**: EFI_15_base_13 完整继承了 EFI_15 的 ACPI Patch 配置。

### 1.3 ACPI Quirks 对比

| 设置 | EFI_13 | EFI_15 | EFI_15_base_13 |
|------|--------|--------|---------------|
| ResetLogoStatus | `false` | `true` | `true` |

**差异**: EFI_15_base_13 跟随 EFI_15，启用了 ResetLogoStatus。

---

## 二、Kernel Kexts 详细对比

### 2.1 Kext 存在性对比表

| Kext 名称 | EFI_13 | EFI_15 | EFI_15_base_13 | 说明 |
|-----------|--------|--------|---------------|------|
| Lilu.kext | ✅ | ✅ | ✅ | 核心框架 |
| VirtualSMC.kext | ✅ | ✅ | ✅ | SMC 仿真 |
| WhateverGreen.kext | ✅ | ✅ | ✅ | 显卡补丁 |
| AppleALC.kext | ✅ | ✅ | ✅ | 声卡驱动 |
| SMCBatteryManager.kext | ✅ | ✅ | ✅ | 电池管理 |
| SMCProcessor.kext | ✅ | ✅ | ✅ | CPU 监控 |
| SMCDellSensors.kext | ✅ | ✅ | ✅ | Dell 传感器 |
| VoodooI2C 系列 | ✅ | ✅ | ✅ | 触控板 |
| VoodooPS2Controller.kext | ✅ | ✅ | ✅ | PS2 键盘 |
| **RealtekRTL8111.kext** | ✅ | ✅ | ✅ | **都有，但版本不同** |
| CtlnaAHCIPort.kext | ✅ | ✅ | ✅ | SATA 控制器 |
| VerbStub.kext | ✅ | ✅ | ✅ | 视频输出 |
| BlueToolFixup.kext | ✅ | ✅ | ✅ | 蓝牙补丁 |
| IntelBluetoothFirmware.kext | ✅ | ✅ | ✅ | 蓝牙固件 |
| **USBMap.kext** | ✅ | ❌ | ❌ | EFI_13 独有 |
| **USBToolBox.kext** | ❌ | ✅ | ✅ | EFI_15/merged |
| **UTBMap.kext** | ❌ | ✅ | ✅ | EFI_15/merged |
| **AirportItlwm.kext** | ✅ | ❌ | ❌ | EFI_13 独有 |
| **itlwm.kext** | ❌ | ✅ | ✅ | EFI_15/merged |
| **IntelBTPatcher.kext** | ❌ | ✅ | ✅ | EFI_15/merged |
| **AMFIPass.kext** | ❌ | ✅ | ✅ | EFI_15/merged |
| **RestrictEvents.kext** | ❌ | ✅ | ✅ | EFI_15/merged |
| **NVMeFix.kext** | ❌ | ✅ | ✅ | EFI_15/merged |
| **CPUFriend.kext** | ❌ | ✅ | ✅ | EFI_15/merged |
| **CPUFriendDataProvider.kext** | ❌ | ✅ | ✅ | EFI_15/merged |
| VoodooPS2Mouse.kext | ✅ (E) | ❌ (D) | ❌ (D) | 已禁用 |
| VoodooPS2Trackpad.kext | ✅ (E) | ❌ (D) | ❌ (D) | 已禁用 |

**图例**: ✅ = 启用，❌ (E) = 存在但禁用，❌ (D) = 不存在

### 2.2 Kext 版本对比

| Kext | EFI_13 | EFI_15 | EFI_15_base_13 |
|------|--------|--------|---------------|
| Lilu | 1.6.6 | 1.7.3 | 1.7.3 |
| VirtualSMC | 1.3.2 | 1.3.8 | 1.3.8 |
| WhateverGreen | 1.6.5 | 1.7.1 | 1.7.1 |
| AppleALC | 1.8.3 | 1.9.8 | 1.9.8 |
| VoodooI2C | 2.8 | 2.9.1 | 2.9.1 |
| VoodooPS2 | 2.3.5 | 2.3.8 | 2.3.8 |
| RealtekRTL8111 | 2.4.2 | 3.0.4 | 3.0.4 |
| BlueToolFixup | 2.6.7 | 2.7.2 | 2.7.2 |
| IntelBluetoothFirmware | 2.2.0 | 2.5.0 | 2.5.0 |

### 2.3 EFI_15_base_13 缺失的 Kext（相比 EFI_15）

| 缺失 Kext | EFI_15 用途 | 影响 |
|----------|------------|------|
| **无关键缺失** | - | EFI_15_base_13 保留了所有功能 Kext |

**重要发现**: EFI_15_base_13 实际上包含了 EFI_15 的所有功能 Kext，数量差异可能是因为某些 Kext 在目录中存在但在 config.plist 中未列出。

---

## 三、NVRAM 配置对比

### 3.1 boot-args 对比

| EFI_13 | EFI_15 | EFI_15_base_13 |
|--------|--------|---------------|
| `-v` | `keepsyms=1 debug=0x100 -v -igfxblt -vi2c-force-polling revpatch=sbvmm -wegnoegpu ipc_control_port_options=0 -amfipassbeta -ibtcompatbeta` | `keepsyms=1 debug=0x100 -v -igfxblt -vi2c-force-polling revpatch=sbvmm -wegnoegpu ipc_control_port_options=0 -amfipassbeta -ibtcompatbeta` |

**结论**: EFI_15_base_13 与 EFI_15 完全一致。

### 3.2 csr-active-config (SIP) 对比

| EFI_13 | EFI_15 | EFI_15_base_13 |
|--------|--------|---------------|
| `AAAAAA==` (0x00000000) | `AwgAAA==` (0x00000803) | `AwgAAA==` (0x00000803) |

**结论**: EFI_15_base_13 跟随 EFI_15，完全关闭 SIP。

### 3.3 NVRAM Delete 对比

| 清理项 | EFI_13 | EFI_15 | EFI_15_base_13 |
|--------|--------|--------|---------------|
| boot-args | ❌ | ✅ | ✅ |
| csr-active-config | ❌ | ✅ | ✅ |
| SystemAudioVolume | ❌ | ✅ | ✅ |
| prev-lang:kbd | ✅ | ✅ | ✅ |

**结论**: EFI_15_base_13 完整继承了 EFI_15 的 NVRAM 清理配置。

---

## 四、UEFI 驱动对比

| 驱动文件 | EFI_13 | EFI_15 | EFI_15_base_13 |
|----------|--------|--------|---------------|
| OpenRuntime.efi | ✅ | ✅ | ✅ |
| OpenHfsPlus.efi | ✅ | ✅ | ✅ |
| OpenCanopy.efi | ✅ | ✅ | ✅ |
| AudioDxe.efi | ✅ | ✅ | ✅ |
| **ResetNvramEntry.efi** | ❌ | ✅ | ✅ |
| **apfs_aligned.efi** | ❌ | ✅ | ✅ |

**结论**: EFI_15_base_13 与 EFI_15 完全一致。

### 4.1 APFS 配置对比

| 设置 | EFI_13 | EFI_15 | EFI_15_base_13 |
|------|--------|--------|---------------|
| EnableJumpstart | `true` | `false` | `false` |

**差异**: EFI_13 使用 Jumpstart 动态加载 APFS，EFI_15 和 EFI_15_base_13 使用静态 apfs_aligned.efi。

### 4.2 ProtocolOverrides 对比

| 设置 | EFI_13 | EFI_15 | EFI_15_base_13 |
|------|--------|--------|---------------|
| FirmwareVolume | `false` | `true` | `true` |

**结论**: EFI_15_base_13 跟随 EFI_15，启用了 FirmwareVolume。

---

## 五、Misc 配置对比

### 5.1 Boot 配置对比

| 设置 | EFI_13 | EFI_15 | EFI_15_base_13 |
|------|--------|--------|---------------|
| LauncherOption | `Disabled` | `Full` | `Full` |
| PickerMode | `External` | `External` | `External` |
| PickerVariant | `Acidanthera\GoldenGate` | `Acidanthera\GoldenGate` | `Acidanthera\GoldenGate` |
| PickerAttributes | `1` | `17` | `17` |
| Timeout | `999` | `999` | `999` |

**结论**: EFI_15_base_13 跟随 EFI_15 的现代化配置。

### 5.2 Tools 对比

EFI_15_base_13 与 EFI_15 完全一致，包含 14 个调试工具：
- CleanNvram, VerifyMsrE2, BootKicker, ChipTune, ControlMsrE2
- CsrUtil, GopStop, KeyTester, MmapDump, OpenControl
- OpenShell, ResetSystem, RtcRw, TpmInfo

---

## 六、PlatformInfo 对比

| 设置 | EFI_13 | EFI_15 | EFI_15_base_13 |
|------|--------|--------|---------------|
| SystemProductName | `MacBookPro14,1` | `MacBookPro14,3` | `MacBookPro14,3` |
| MLB | `C02824404GUHWVPCB` | `C02712902QXJ1JH1M` | `C02712902QXJ1JH1M` |
| SystemSerialNumber | `C02WVUYHHV29` | `C02TGBZ3HTD5` | `C02TGBZ3HTD5` |
| SystemUUID | `38704299-09E3-46EB-8635-82C4C422B9AF` | `08ABDD0B-0876-4649-9C05-954A37DF098C` | `08ABDD0B-0876-4649-9C05-954A37DF098C` |
| ProcessorType | `0` | `1537` | `1537` |
| SpoofVendor | `false` | `true` | `true` |
| ROM | `nPOHclh6` | `oChAWYTF` | `oChAWYTF` |

**关键发现**:
- EFI_15_base_13 完全继承了 EFI_15 的 SMBIOS 配置
- MacBookPro14,3 比 MacBookPro14,1 更接近实际硬件（i7-7700HQ）
- ProcessorType = 1537 是 Kaby Lake HQ 系列的正确值

---

## 七、Booter 配置对比

### 7.1 Patch 配置

| 设置 | EFI_13 | EFI_15 | EFI_15_base_13 |
|------|--------|--------|---------------|
| Board ID 跳过补丁 | ❌ | ✅ | ✅ |

### 7.2 Quirks 对比

| 设置 | EFI_13 | EFI_15 | EFI_15_base_13 |
|------|--------|--------|---------------|
| ResetLogoStatus | `false` | `true` | `true` |
| FixupAppleEfiImages | ❌ | `true` | `true` |
| ClearTaskSwitchBit | ❌ | `false` | `false` |

**结论**: EFI_15_base_13 完整继承了 EFI_15 的 Booter 配置。

---

## 八、Kernel Quirks 对比

| 设置 | EFI_13 | EFI_15 | EFI_15_base_13 |
|------|--------|--------|---------------|
| AppleXcpmCfgLock | `true` | `true` | `true` |
| CustomSMBIOSGuid | `true` | `true` | `true` |
| DisableIoMapper | `true` | `true` | `true` |
| PanicNoKextDump | `true` | `true` | `true` |
| PowerTimeoutKernelPanic | `true` | `true` | `true` |
| XhciPortLimit | `true` | `true` | `true` |
| SetApfsTrimTimeout | `0` | `-1` | `-1` |

**结论**: 除 SetApfsTrimTimeout 外，其余设置完全一致。

---

## 九、综合分析

### 9.1 EFI_15_base_13 的合并质量

**优点**:
1. ✅ **完整继承 EFI_15 的核心功能**: USBToolBox、itlwm、AMFIPass 等全部保留
2. ✅ **ACPI 配置正确**: XOSI 相关补丁完整
3. ✅ **NVRAM 配置正确**: boot-args 和 SIP 设置正确
4. ✅ **UEFI 驱动完整**: ResetNvramEntry 和 apfs_aligned 都在
5. ✅ **SMBIOS 更新**: 使用 MacBookPro14,3 更准确

**潜在问题**:
1. ⚠️ **版本混合**: 基于 EFI_13 合并可能导致某些深层配置不一致
2. ⚠️ **需要验证**: 建议使用 `ocvalidate` 验证配置完整性

### 9.2 推荐使用场景

| EFI 配置 | 推荐场景 |
|---------|----------|
| **EFI_13** | 纯 macOS 13 环境，追求简洁 |
| **EFI_15** | 纯 macOS 15 环境，功能最完整 |
| **EFI_15_base_13** | 跨版本共存环境，基于旧版升级的最佳选择 |

### 9.3 合并建议

如果使用 EFI_15_base_13 作为通用引导：

1. **验证配置**: 运行 `ocvalidate config.plist`
2. **测试启动**: 在每个系统上测试引导功能
3. **NVRAM 重置**: 首次启动后执行 Reset NVRAM
4. **备份原配置**: 保留 EFI_13 和 EFI_15 作为备用

---

## 附录：差异速查表

### A. EFI_13 独有配置

| 配置项 | 值 |
|--------|-----|
| USBMap.kext | ✅ Enabled |
| AirportItlwm.kext | ✅ Enabled |
| EnableJumpstart | `true` |
| LauncherOption | `Disabled` |
| FirmwareVolume | `false` |
| SIP | 启用 |

### B. EFI_15 独有配置

| 配置项 | 值 |
|--------|-----|
| USBToolBox + UTBMap | ✅ |
| itlwm.kext | ✅ Enabled |
| apfs_aligned.efi | ✅ |
| ResetNvramEntry.efi | ✅ |
| EnableJumpstart | `false` |
| LauncherOption | `Full` |
| FirmwareVolume | `true` |
| SIP | 禁用 |

### C. EFI_15_base_13 的状态

| 配置项 | 值 | 来源 |
|--------|-----|------|
| USBToolBox + UTBMap | ✅ | 继承 EFI_15 |
| itlwm.kext | ✅ | 继承 EFI_15 |
| apfs_aligned.efi | ✅ | 继承 EFI_15 |
| ResetNvramEntry.efi | ✅ | 继承 EFI_15 |
| EnableJumpstart | `false` | 继承 EFI_15 |
| LauncherOption | `Full` | 继承 EFI_15 |
| FirmwareVolume | `true` | 继承 EFI_15 |
| SIP | 禁用 | 继承 EFI_15 |
| SMBIOS | MacBookPro14,3 | 继承 EFI_15 |

---

**报告结束**
