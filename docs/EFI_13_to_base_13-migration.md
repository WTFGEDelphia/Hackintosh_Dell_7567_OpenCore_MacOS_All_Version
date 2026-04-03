# EFI_13 到 EFI_15_base_13 迁移差异分析报告

> **生成时间**: 2026-04-03
>
> **分析对象**:
> - **源配置**: `EFI_13/EFI/OC/config.plist` (macOS 13 Ventura 专用)
> - **目标配置**: `EFI_15_base_13/OC/config.plist` (基于 EFI_13 手动合并的跨版本配置)
>
> **背景说明**: EFI_15_base_13 是用户根据 EFI_15 作为参考，手动合并到 EFI_13 基础上的配置，旨在实现 macOS 13 和 macOS 15 的跨版本共存。

---

## 一、核心变更概览

| 配置类别 | EFI_13 (源) | EFI_15_base_13 (目标) | 变更类型 |
|---------|-------------|----------------------|---------|
| **Kext 数量** | 19 个 | 24 个 | +5 个新增 |
| **SSDT 数量** | 5 个 | 6 个 | +1 个 (SSDT-XOSI) |
| **UEFI 驱动** | 4 个 | 6 个 | +2 个 (ResetNvramEntry, apfs_aligned) |
| **SIP 状态** | 启用 | 禁用 | 重大变更 |
| **USB 映射方案** | USBMap.kext | USBToolBox + UTBMap | 架构重构 |
| **WiFi 驱动** | AirportItlwm | itlwm | 驱动替换 |
| **SMBIOS 机型** | MacBookPro14,1 | MacBookPro14,3 | 机型更新 |
| **LauncherOption** | Disabled | Full | 引导策略变更 |

---

## 二、ACPI 配置变更详情

### 2.1 SSDT 添加列表

| 顺序 | SSDT 名称 | EFI_13 | EFI_15_base_13 | 变更说明 |
|------|----------|--------|---------------|---------|
| 1 | SSDT-XOSI.aml | ❌ | ✅ | **新增** - _OSI 方法替换 |
| 2 | SSDT-PLUG.aml | ✅ | ✅ | 保持不变 |
| 3 | SSDT-PNLF.aml | ✅ | ✅ | 保持不变 |
| 4 | SSDT-EC-USBX-LAPTOP.aml | ✅ | ✅ | 保持不变 |
| 5 | SSDT-GPI0.aml | ✅ | ✅ | 保持不变 |
| 6 | SSDT-DXTR.aml | ✅ | ✅ | 保持不变 |

### 2.2 ACPI Patch 变更

| Patch 名称 | EFI_13 | EFI_15_base_13 | Find (Base64) | 说明 |
|------------|--------|---------------|---------------|------|
| BRT6 to XRT6 | ✅ | ✅ | `QlJUNgKgC5M=` → `WFJUNgKgC5M=` | 保持不变 |
| Gprw to Xprw | ✅ | ✅ | `R1BSVwI=` → `WFBSVwI=` | 保持不变 |
| **PNLF to XNLF Rename** | ❌ | ✅ | `UE5MRg==` → `WE5MRg==` | **新增** |
| **_OSI to XOSI rename** | ❌ | ✅ | `_OSI` → `_OSI` | **新增** |

### 2.3 ACPI Quirks 变更

| 设置项 | EFI_13 | EFI_15_base_13 | 说明 |
|--------|--------|---------------|------|
| FadtEnableReset | `false` | `false` | 无变更 |
| NormalizeHeaders | `false` | `false` | 无变更 |
| RebaseRegions | `false` | `false` | 无变更 |
| ResetHwSig | `false` | `false` | 无变更 |
| **ResetLogoStatus** | `false` | `true` | **已修改** |
| SyncTableIds | `false` | `false` | 无变更 |

---

## 三、Boot er 配置变更详情

### 3.1 Booter Patch (新增)

**EFI_13**: 无 Booter Patch

**EFI_15_base_13**: 新增 Board ID 检查跳过补丁
```xml
<Patch>
    <Arch>x86_64</Arch>
    <Comment>Skip Board ID check</Comment>
    <Count>0</Count>
    <Enabled>true</Enabled>
    <Find>AFAAbABhAHQAZgBvAHIAbQBTAHUAcABwAG8AcgB0AC4AcABsAGkAcwB0</Find>
    <Identifier>Apple</Identifier>
    <Replace>AC4ALgAuAC4ALgAuAC4ALgAuAC4ALgAuAC4ALgAuAC4ALgAuAC4ALgAu</Replace>
</Patch>
```

### 3.2 Booter Quirks 变更

| 设置项 | EFI_13 | EFI_15_base_13 | 说明 |
|--------|--------|---------------|------|
| AllowRelocationBlock | `false` | `false` | 无变更 |
| AvoidRuntimeDefrag | `true` | `true` | 无变更 |
| DevirtualiseMmio | `false` | `false` | 无变更 |
| DisableSingleUser | `false` | `false` | 无变更 |
| EnableSafeModeSlide | `true` | `true` | 无变更 |
| EnableWriteUnprotector | `true` | `true` | 无变更 |
| **FixupAppleEfiImages** | 不存在 | `true` | **新增** |
| ForceBooterSignature | `false` | `false` | 无变更 |
| ProvideCustomSlide | `true` | `true` | 无变更 |
| ResizeAppleGpuBars | `-1` | `-1` | 无变更 |
| SetupVirtualMap | `true` | `true` | 无变更 |
| SyncRuntimePermissions | `false` | `false` | 无变更 |

---

## 四、Kernel Kexts 变更详情

### 4.1 新增 Kext (5 个)

| Kext 名称 | 版本 | 用途 | 必要性 |
|----------|------|------|--------|
| **USBToolBox.kext** | V1.2.0 | USB 映射引擎 | **必需** |
| **UTBMap.kext** | V1.1 | USB 映射配置 | **必需** |
| **AMFIPass.kext** | V1.4.1 | AMFI 绕过 | **必需** (跨版本) |
| **RestrictEvents.kext** | V1.1.7 | 机型欺骗/事件限制 | 推荐 |
| **NVMeFix.kext** | V1.1.4 | NVMe 电源管理 | 推荐 |
| **CPUFriend.kext** | V1.2.3 | CPU 电源管理优化 | 推荐 |
| **CPUFriendDataProvider.kext** | V1.0.0 | CPUFriend 数据源 | 配套 |
| **IntelBTPatcher.kext** | V2.5.0 | Intel 蓝牙补丁 | 推荐 |
| **itlwm.kext** | V2.3.0 | WiFi 驱动 | **必需** (替换 AirportItlwm) |

### 4.2 移除/禁用 Kext

| Kext 名称 | EFI_13 状态 | EFI_15_base_13 状态 | 说明 |
|----------|-------------|---------------------|------|
| **USBMap.kext** | ✅ Enabled | ❌ 移除 | 被 USBToolBox 替代 |
| **AirportItlwm.kext** | ✅ Enabled | ❌ 移除 | 被 itlwm 替代 |
| **IntelBluetoothFirmware.kext** | ✅ Enabled | ✅ Enabled | 保留，版本升级 2.2.0 → 2.5.0 |
| **VoodooPS2Mouse.kext** | ✅ Enabled | ❌ Disabled | 禁用 (触控板冲突) |
| **VoodooPS2Trackpad.kext** | ✅ Enabled | ❌ Disabled | 禁用 (I2C 冲突) |

### 4.3 Kext 版本升级

| Kext | EFI_13 | EFI_15_base_13 | 升级说明 |
|------|--------|---------------|---------|
| Lilu | 1.6.6 | 1.7.3 | 核心框架更新 |
| VirtualSMC | 1.3.2 | 1.3.8 | SMC 仿真更新 |
| WhateverGreen | 1.6.5 | 1.7.1 | 显卡补丁更新 |
| AppleALC | 1.8.3 | 1.9.8 | 声卡驱动更新 |
| VoodooI2C | 2.8 | 2.9.1 | 触控板驱动更新 |
| VoodooPS2 | 2.3.5 | 2.3.8 | PS2 驱动更新 |
| RealtekRTL8111 | 2.4.2 | 3.0.4 | 网卡驱动大版本更新 |
| BlueToolFixup | 2.6.7 | 2.7.2 | 蓝牙补丁更新 |
| IntelBluetoothFirmware | 2.2.0 | 2.5.0 | 蓝牙固件更新 |
| SMC 系列插件 | 1.3.2 | 1.3.8 | 统一版本更新 |

### 4.4 Kext 加载顺序变更

**EFI_13 顺序**:
```
1. USBMap.kext (静态 USB 映射)
2. CtlnaAHCIPort.kext
3. Lilu.kext (核心框架)
4. VirtualSMC.kext
5. WhateverGreen.kext
6. AppleALC.kext
...
17. AirportItlwm.kext
```

**EFI_15_base_13 顺序**:
```
1. Lilu.kext (核心框架优先)
2. VirtualSMC.kext
3. CtlnaAHCIPort.kext
4. WhateverGreen.kext
5. AppleALC.kext
...
23. USBToolBox.kext (USB 引擎)
24. UTBMap.kext (USB 配置)
...
27. itlwm.kext (WiFi)
```

**关键变更**: USBToolBox.kext 必须在 UTBMap.kext 之前加载

---

## 五、NVRAM 配置变更详情

### 5.1 boot-args 变更

**EFI_13**:
```
-v
```

**EFI_15_base_13**:
```
keepsyms=1 debug=0x100 -v -igfxblt -vi2c-force-polling revpatch=sbvmm -wegnoegpu ipc_control_port_options=0 -amfipassbeta -ibtcompatbeta
```

**新增参数详解**:

| 参数 | 作用 | 必要性 |
|------|------|--------|
| `keepsyms=1` | 崩溃时显示符号信息 | 推荐 |
| `debug=0x100` | 启用调试日志 | 推荐 |
| `-igfxblt` | Intel 显卡亮度控制修复 | **必需** |
| `-vi2c-force-polling` | I2C 触控板强制轮询模式 | 推荐 |
| `revpatch=sbvmm` | OCLP 补丁策略 | **必需** (跨版本) |
| `-wegnoegpu` | 禁用独显 (GTX 1050/1050Ti) | **必需** |
| `ipc_control_port_options=0` | 修复 Electron 应用崩溃 | 推荐 |
| `-amfipassbeta` | AMFIPass 跨版本生效 | **必需** |
| `-ibtcompatbeta` | Intel 蓝牙初始化修复 | **必需** (macOS 15) |

### 5.2 csr-active-config (SIP) 变更

| EFI_13 | EFI_15_base_13 | 说明 |
|--------|---------------|------|
| `AAAAAA==` (0x00000000) | `AwgAAA==` (0x00000803) | 完全关闭 SIP |

**0x00000803 含义**:
- CSR_VALID_APPLE_OS (0x00000001)
- CSR_ALLOW_UNTRUSTED_KEXTS (0x00000002)
- CSR_ALLOW_UNRESTRICTED_FS (0x00000004)
- CSR_ALLOW_TASK_FOR_PID (0x00000008)
- CSR_ALLOW_UNRESTRICTED_DARWIN_KEXTS (0x00000800)

### 5.3 NVRAM Delete 变更

| 清理项 | EFI_13 | EFI_15_base_13 | 说明 |
|--------|--------|---------------|------|
| boot-args | ❌ | ✅ | **新增** - 防止旧参数覆盖 |
| csr-active-config | ❌ | ✅ | **新增** |
| SystemAudioVolume | ❌ | ✅ | **新增** |
| prev-lang:kbd | ✅ | ✅ | 保持不变 |
| bluetoothExternalDongleFailed | ❌ | ✅ | **新增** |
| bluetoothInternalControllerInfo | ❌ | ✅ | **新增** |
| ForceDisplayRotationInEFI | ❌ | ✅ | **新增** |
| revpatch | ❌ | ✅ | **新增** |
| rtc-blacklist | ❌ | ✅ | **新增** |

### 5.4 新增 NVRAM Add 项

**4D1FDA02-38C7-4A6A-9CC6-4BCCA8B30102 (OCLM GUID)**:
```xml
<key>revpatch</key>
<string>sbvmm</string>
```

---

## 六、UEFI 驱动变更详情

### 6.1 驱动列表变更

| 驱动文件 | EFI_13 | EFI_15_base_13 | 说明 |
|----------|--------|---------------|------|
| OpenRuntime.efi | ✅ | ✅ | 保持不变 |
| OpenHfsPlus.efi | ✅ | ✅ | 保持不变 |
| OpenCanopy.efi | ✅ | ✅ | 保持不变 |
| AudioDxe.efi | ✅ | ✅ | 保持不变 |
| **ResetNvramEntry.efi** | ❌ | ✅ | **新增** |
| **apfs_aligned.efi** | ❌ | ✅ | **新增** |

### 6.2 APFS 配置变更

| 设置项 | EFI_13 | EFI_15_base_13 | 说明 |
|--------|--------|---------------|------|
| EnableJumpstart | `true` | `false` | 改用静态驱动 |
| GlobalConnect | `false` | `false` | 无变更 |
| HideVerbose | `true` | `true` | 无变更 |
| JumpstartHotPlug | `false` | `false` | 无变更 |
| MinDate | `-1` | `-1` | 无变更 |
| MinVersion | `-1` | `-1` | 无变更 |

### 6.3 ProtocolOverrides 变更

| 设置项 | EFI_13 | EFI_15_base_13 | 说明 |
|--------|--------|---------------|------|
| AppleAudio | `false` | `false` | 无变更 |
| ... | ... | ... | ... |
| **FirmwareVolume** | `false` | `true` | **已修改** - 必需用于 macOS 15 |

### 6.4 UEFI Quirks 变更

| 设置项 | EFI_13 | EFI_15_base_13 | 说明 |
|--------|--------|---------------|------|
| ReleaseUsbOwnership | `true` | `true` | 无变更 |
| RequestBootVarRouting | `true` | `true` | 无变更 |
| **ShimRetainProtocol** | 不存在 | `true` | **新增** |

---

## 七、Misc 配置变更详情

### 7.1 Boot 配置变更

| 设置项 | EFI_13 | EFI_15_base_13 | 说明 |
|--------|--------|---------------|------|
| ConsoleAttributes | `0` | `0` | 无变更 |
| HibernateMode | `None` | `None` | 无变更 |
| **LauncherOption** | `Disabled` | `Full` | **已修改** - 写入 UEFI 启动项 |
| LauncherPath | `Default` | `Default` | 无变更 |
| **PickerAttributes** | `1` | `17` | **已修改** - 启用更多图形选项 |
| PickerAudioAssist | `false` | `false` | 无变更 |
| **PickerMode** | `External` | `External` | 无变更 |
| Timeout | `999` | `999` | 无变更 |

### 7.2 Debug 配置变更

| 设置项 | EFI_13 | EFI_15_base_13 | 说明 |
|--------|--------|---------------|------|
| AppleDebug | `false` | `false` | 无变更 |
| ApplePanic | `false` | `false` | 无变更 |
| DisableWatchDog | `true` | `true` | 无变更 |
| DisplayDelay | `0` | `0` | 无变更 |
| DisplayLevel | `2147483650` | `2147483650` | 无变更 |
| **LogModules** | `""` | `"*"` | **已修改** - 记录所有模块 |

### 7.3 Tools 变更

**EFI_13 Tools (3 个)**:
- CleanNvram (Enabled)
- VerifyMsrE2 (Disabled)
- modGRUBShell (Disabled)

**EFI_15_base_13 Tools (14 个)**:
- CleanNvram (Enabled)
- VerifyMsrE2 (Disabled)
- **BootKicker.efi (Enabled)** - 新增
- **ChipTune.efi (Enabled)** - 新增
- **ControlMsrE2.efi (Enabled)** - 新增
- **CsrUtil.efi (Enabled)** - 新增
- **GopStop.efi (Enabled)** - 新增
- **KeyTester.efi (Enabled)** - 新增
- **MmapDump.efi (Enabled)** - 新增
- **OpenControl.efi (Enabled)** - 新增
- **OpenShell.efi (Enabled)** - 新增
- **ResetSystem.efi (Enabled)** - 新增
- **RtcRw.efi (Enabled)** - 新增
- **TpmInfo.efi (Enabled)** - 新增

---

## 八、PlatformInfo 配置变更详情

### 8.1 Generic 配置变更

| 设置项 | EFI_13 | EFI_15_base_13 | 说明 |
|--------|--------|---------------|------|
| AdviseFeatures | `false` | `false` | 无变更 |
| **MLB** | C02824404GUHWVPCB | C02712902QXJ1JH1M | 已修改 |
| MaxBIOSVersion | `false` | `false` | 无变更 |
| **ProcessorType** | 0 | 1537 | **已修改** - Kaby Lake HQ |
| **ROM** | nPOHclh6 | oChAWYTF | 已修改 |
| **SpoofVendor** | `false` | `true` | **已修改** |
| SystemMemoryStatus | `Auto` | `Auto` | 无变更 |
| **SystemProductName** | MacBookPro14,1 | MacBookPro14,3 | **已修改** |
| **SystemSerialNumber** | C02WVUYHHV29 | C02TGBZ3HTD5 | 已修改 |
| **SystemUUID** | 38704299-09E3-46EB-8635-82C4C422B9AF | 08ABDD0B-0876-4649-9C05-954A37DF098C | 已修改 |

**机型变更说明**:
- MacBookPro14,1: i5-7300HQ + HD 630 + 8GB
- MacBookPro14,3: i7-7700HQ + HD 630 + 16GB

---

## 九、关键变更总结

### 9.1 必须执行的变更

| 变更项 | 原因 | 风险等级 |
|--------|------|---------|
| SIP 关闭 | OCLP 补丁必需 | ⚠️ 安全影响 |
| USBToolBox 迁移 | 跨版本 USB 兼容性 | ✅ 改进 |
| itlwm 替换 AirportItlwm | 避免版本绑定问题 | ✅ 改进 |
| boot-args 更新 | 支持跨版本启动 | ✅ 必需 |
| FirmwareVolume 启用 | 显示 macOS 15 安装盘 | ✅ 必需 |
| LauncherOption = Full | 抵抗 Windows 启动项覆盖 | ✅ 推荐 |

### 9.2 可选变更

| 变更项 | 原因 | 推荐度 |
|--------|------|--------|
| SMBIOS 升级 | 更准确反映硬件 | ⭐⭐⭐⭐ |
| CPUFriend | 优化 CPU 电源管理 | ⭐⭐⭐ |
| NVMeFix | 优化 NVMe 功耗 | ⭐⭐⭐ |
| 调试工具 | 便于问题排查 | ⭐⭐⭐⭐ |

### 9.3 需要手动验证的配置

| 配置项 | 验证方法 | 预期结果 |
|--------|---------|---------|
| USB 映射 | 系统信息 → USB | 所有端口速度正常 |
| 触控板 | 系统偏好设置 | 多指手势正常 |
| 声卡 | 系统偏好设置 → 声音 | 输出/输入正常 |
| 亮度调节 | Fn 键 + 亮度键 | 亮度可调节 |
| 睡眠/唤醒 | 合盖/开盖 | 正常睡眠唤醒 |

---

## 十、迁移检查清单

### 10.1 文件层面

- [ ] 将 USBToolBox.kext 和 UTBMap.kext 复制到 EFI/OC/Kexts
- [ ] 删除 USBMap.kext
- [ ] 添加 AMFIPass.kext、RestrictEvents.kext、NVMeFix.kext
- [ ] 添加 CPUFriend.kext 和 CPUFriendDataProvider.kext
- [ ] 添加 IntelBTPatcher.kext
- [ ] 将 AirportItlwm.kext 替换为 itlwm.kext
- [ ] 添加 SSDT-XOSI.aml 到 EFI/OC/ACPI
- [ ] 添加 ResetNvramEntry.efi 和 apfs_aligned.efi 到 EFI/OC/Drivers

### 10.2 config.plist 层面

- [ ] 更新 ACPI Patch (添加 PNLF 和 XOSI 重命名)
- [ ] 更新 Kext 加载顺序 (Lilu 优先，USBToolBox 在 UTBMap 之前)
- [ ] 更新 boot-args
- [ ] 更新 csr-active-config
- [ ] 更新 NVRAM Delete 列表
- [ ] 更新 UEFI Drivers 列表
- [ ] 启用 FirmwareVolume ProtocolOverride
- [ ] 更新 Misc Boot 配置 (LauncherOption, PickerAttributes)
- [ ] 更新 PlatformInfo (SMBIOS 信息)

### 10.3 验证步骤

1. **配置验证**: `ocvalidate config.plist`
2. **NVRAM 重置**: 在 OpenCore 引导界面选择 Reset NVRAM
3. **系统测试**: 分别测试 macOS 13 和 macOS 15
4. **硬件测试**: 按照 9.3 验证关键硬件功能

---

## 附录：变更配置文件清单

### A. 新增文件 (24 个)

**Kexts**:
- USBToolBox.kext
- UTBMap.kext
- AMFIPass.kext
- RestrictEvents.kext
- NVMeFix.kext
- CPUFriend.kext
- CPUFriendDataProvider.kext
- IntelBTPatcher.kext
- itlwm.kext

**ACPI**:
- SSDT-XOSI.aml

**Drivers**:
- ResetNvramEntry.efi
- apfs_aligned.efi

**Tools**:
- BootKicker.efi
- ChipTune.efi
- ControlMsrE2.efi
- CsrUtil.efi
- GopStop.efi
- KeyTester.efi
- MmapDump.efi
- OpenControl.efi
- OpenShell.efi
- ResetSystem.efi
- RtcRw.efi
- TpmInfo.efi

### B. 删除文件 (2 个)

- USBMap.kext
- AirportItlwm.kext

### C. 更新文件 (版本升级)

- Lilu.kext (1.6.6 → 1.7.3)
- VirtualSMC.kext (1.3.2 → 1.3.8)
- WhateverGreen.kext (1.6.5 → 1.7.1)
- AppleALC.kext (1.8.3 → 1.9.8)
- VoodooI2C.kext (2.8 → 2.9.1)
- VoodooPS2Controller.kext (2.3.5 → 2.3.8)
- RealtekRTL8111.kext (2.4.2 → 3.0.4)
- BlueToolFixup.kext (2.6.7 → 2.7.2)
- IntelBluetoothFirmware.kext (2.2.0 → 2.5.0)
- SMC* 插件系列 (1.3.2 → 1.3.8)

---

**报告结束**
