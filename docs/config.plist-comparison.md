# Dell 7567 OpenCore config.plist 深度对比报告

> **生成时间**: 2026-04-03
>
> **对比文件**:
>
> - `EFI_13/EFI/OC/config.plist` (macOS 13 Ventura)
> - `EFI_15/EFI/OC/config.plist` (macOS 15 Sequoia)

---

## 执行摘要

| 对比维度                | EFI_13 (macOS 13)     | EFI_15 (macOS 15)                               |
| ----------------------- | --------------------- | ----------------------------------------------- |
| **OpenCore 版本** |    0.9.3              |    1.0.7                                        |
| **Kext 数量**     | 19 个                 | 27 个                                           |
| **SSDT 数量**     | 5 个                  | 6 个 (多 SSDT-XOSI)                             |
| **UEFI 驱动**     | 4 个                  | 6 个 (多 ResetNvramEntry.efi, apfs_aligned.efi) |
| **SIP 状态**      | 完全启用 (0x00000000) | 完全禁用 (0x00000803)                           |
| **SMBIOS 机型**   | MacBookPro14,1        | MacBookPro14,3                                  |

---

## 一、ACPI 配置对比

### 1.1 SSDT 加载列表

| SSDT 文件               | EFI_13 | EFI_15 | 说明                                 |
| ----------------------- | ------ | ------ | ------------------------------------ |
| SSDT-PLUG.aml           | ✅     | ✅     | CPU 电源管理                         |
| SSDT-PNLF.aml           | ✅     | ✅     | 亮度控制                             |
| SSDT-EC-USBX-LAPTOP.aml | ✅     | ✅     | 嵌入式控制器                         |
| SSDT-GPI0.aml           | ✅     | ✅     | I2C 触控板                           |
| SSDT-DXTR.aml           | ✅     | ✅     | 深度睡眠                             |
| **SSDT-XOSI.aml** | ❌     | ✅     | _OSI 方法替换**(EFI_15 新增)** |

### 1.2 ACPI Patch 对比

| Patch 名称                    | EFI_13 | EFI_15 | 说明                                    |
| ----------------------------- | ------ | ------ | --------------------------------------- |
| BRT6 to XRT6                  | ✅     | ✅     | 亮度键修复                              |
| Gprw to Xprw                  | ✅     | ✅     | 睡眠修复                                |
| **PNLF to XNLF Rename** | ❌     | ✅     | 亮度 SSDT 重命名**(EFI_15 新增)** |
| **_OSI to XOSI rename** | ❌     | ✅     | Windows 检测绕过**(EFI_15 新增)** |

**关键发现**: EFI_15 新增了 XOSI 相关补丁，这是为了在新版 macOS 中正确绕过 Windows 平台检测。

---

## 二、Kernel Kexts 对比

### 2.1 Kext 加载列表对比

| Kext 名称                | EFI_13 | EFI_15 | 版本差异       | 说明          |
| ------------------------ | ------ | ------ | -------------- | ------------- |
| Lilu.kext                | ✅     | ✅     | 1.6.6 → 1.7.3 | 核心插件框架  |
| VirtualSMC.kext          | ✅     | ✅     | 1.3.2 → 1.3.8 | SMC 仿真      |
| WhateverGreen.kext       | ✅     | ✅     | 1.6.5 → 1.7.1 | 显卡补丁      |
| AppleALC.kext            | ✅     | ✅     | 1.8.3 → 1.9.8 | 声卡驱动      |
| SMCBatteryManager.kext   | ✅     | ✅     | 1.3.2 → 1.3.8 | 电池管理      |
| SMCProcessor.kext        | ✅     | ✅     | 1.3.2 → 1.3.8 | CPU 监控      |
| SMCDellSensors.kext      | ✅     | ✅     | 1.3.2 → 1.3.8 | Dell 传感器   |
| VoodooI2C 系列           | ✅     | ✅     | 2.8 → 2.9.1   | 触控板        |
| VoodooPS2Controller.kext | ✅     | ✅     | 2.3.5 → 2.3.8 | PS2 键盘/鼠标 |
| RealtekRTL8111.kext      | ✅     | ✅     | 2.4.2 → 3.0.4 | 有线网卡      |
| CtlnaAHCIPort.kext       | ✅     | ✅     | 341.0.2 (同)   | SATA 控制器   |
| VerbStub.kext            | ✅     | ✅     | 1.0.3 (同)     | 视频输出      |
| BlueToolFixup.kext       | ✅     | ✅     | 2.6.7 → 2.7.2 | 蓝牙补丁      |

### 2.2 关键差异 Kext

| Kext 名称                             | EFI_13     | EFI_15      | 说明                           |
| ------------------------------------- | ---------- | ----------- | ------------------------------ |
| **USBMap.kext**                 | ✅ Enabled | ❌ 移除     | 旧版 USB 映射方案              |
| **USBToolBox.kext**             | ❌         | ✅ Enabled  | 新版 USB 映射引擎              |
| **UTBMap.kext**                 | ❌         | ✅ Enabled  | USBToolBox 映射配置            |
| **AirportItlwm.kext**           | ✅ Enabled | ❌ 移除     | 旧版 WiFi 驱动 (版本绑定)      |
| **itlwm.kext**                  | ❌         | ✅ Enabled  | 新版 WiFi 驱动 (通用)          |
| **IntelBluetoothFirmware.kext** | ✅         | ✅          | 蓝牙固件 (版本 2.2.0 → 2.5.0) |
| **IntelBTPatcher.kext**         | ❌         | ✅ Enabled  | 蓝牙补丁 (配合 IBF 使用)       |
| **AMFIPass.kext**               | ❌         | ✅ Enabled  | AMFI 绕过 (macOS 15 必需)      |
| **RestrictEvents.kext**         | ❌         | ✅ Enabled  | 事件限制 (机型欺骗)            |
| **NVMeFix.kext**                | ❌         | ✅ Enabled  | NVMe 电源管理                  |
| **CPUFriend.kext**              | ❌         | ✅ Enabled  | CPU 电源管理优化               |
| **CPUFriendDataProvider.kext**  | ❌         | ✅ Enabled  | CPUFriend 数据                 |
| **VoodooPS2Mouse.kext**         | ✅ Enabled | ❌ Disabled | PS2 鼠标 (触控板冲突)          |
| **VoodooPS2Trackpad.kext**      | ✅ Enabled | ❌ Disabled | PS2 触控板 (I2C 冲突)          |

### 2.3 Kext 排序差异

**EFI_13 关键顺序**:

```
1. USBMap.kext          (静态 USB 映射)
2. CtlnaAHCIPort.kext
3. Lilu.kext            (核心框架)
4. VirtualSMC.kext
5. WhateverGreen.kext
6. AppleALC.kext
...
17. AirportItlwm.kext   (WiFi - 版本绑定)
```

**EFI_15 关键顺序**:

```
1. Lilu.kext            (核心框架优先)
2. VirtualSMC.kext
3. CtlnaAHCIPort.kext
4. WhateverGreen.kext
5. AppleALC.kext
...
23. USBToolBox.kext     (USB 引擎)
24. UTBMap.kext         (USB 配置)
...
28. itlwm.kext          (WiFi - 通用版)
```

**关键发现**: EFI_15 明确将 USBToolBox.kext 放在 UTBMap.kext 之前，这是引擎必须在配置之前加载的要求。

---

## 三、NVRAM 配置对比

### 3.1 boot-args 对比

| EFI_13 | EFI_15                                                                                                                                       |
| ------ | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `-v` | `keepsyms=1 debug=0x100 -v -igfxblt -vi2c-force-polling revpatch=sbvmm -wegnoegpu ipc_control_port_options=0 -amfipassbeta -ibtcompatbeta` |

**EFI_15 新增参数详解**:

| 参数                           | 作用                                 |
| ------------------------------ | ------------------------------------ |
| `keepsyms=1`                 | 崩溃时显示符号信息                   |
| `debug=0x100`                | 启用调试日志                         |
| `-igfxblt`                   | Intel 显卡亮度控制修复               |
| `-vi2c-force-polling`        | I2C 触控板强制轮询模式               |
| `revpatch=sbvmm`             | OCLP 补丁策略                        |
| `-wegnoegpu`                 | **禁用独显** (GTX 1050/1050Ti) |
| `ipc_control_port_options=0` | 修复 Electron 应用崩溃 (VSCode 等)   |
| `-amfipassbeta`              | AMFIPass 跨版本生效                  |
| `-ibtcompatbeta`             | Intel 蓝牙初始化修复 (macOS 15 必需) |

### 3.2 csr-active-config (SIP) 对比

| EFI_13                    | EFI_15                    |
| ------------------------- | ------------------------- |
| `AAAAAA==` (0x00000000) | `AwgAAA==` (0x00000803) |

**说明**:

- EFI_13: SIP 完全启用
- EFI_15: SIP 完全禁用 (`0x00000803` = `CSR_VALID_APPLE_OS | CSR_ALLOW_UNTRUSTED_KEXTS | CSR_ALLOW_UNRESTRICTED_FS | CSR_ALLOW_TASK_FOR_PID`)

### 3.3 NVRAM Delete 对比

EFI_15 新增了对以下项的清理:

- `boot-args`
- `csr-active-config`
- `SystemAudioVolume`
- `bluetoothExternalDongleFailed`
- `bluetoothInternalControllerInfo`
- `ForceDisplayRotationInEFI`

**重要性**: 不清理 NVRAM 会导致旧配置覆盖新配置，是"修改不生效"的常见原因。

---

## 四、UEFI 驱动对比

| 驱动文件                      | EFI_13 | EFI_15 | 说明                                  |
| ----------------------------- | ------ | ------ | ------------------------------------- |
| OpenRuntime.efi               | ✅     | ✅     | 内存管理                              |
| OpenHfsPlus.efi               | ✅     | ✅     | HFS+ 读取                             |
| OpenCanopy.efi                | ✅     | ✅     | 图形引导                              |
| AudioDxe.efi                  | ✅     | ✅     | 音频支持                              |
| **ResetNvramEntry.efi** | ❌     | ✅     | NVRAM 清理 (macOS 15 维护必备)        |
| **apfs_aligned.efi**    | ❌     | ✅     | APFS 驱动 (可被 EnableJumpstart 替代) |

### 4.1 APFS 配置对比

| 设置            | EFI_13   | EFI_15    |
| --------------- | -------- | --------- |
| EnableJumpstart | `true` | `false` |

**说明**: EFI_13 使用 Jumpstart 动态抓取 APFS 驱动，EFI_15 使用静态 `apfs_aligned.efi`。

### 4.2 ProtocolOverrides 对比

| 设置           | EFI_13    | EFI_15   |
| -------------- | --------- | -------- |
| FirmwareVolume | `false` | `true` |

**关键**: EFI_15 必须启用 `FirmwareVolume`，否则 macOS 15+ 的安装盘不会显示在引导菜单中。

---

## 五、Misc 配置对比

### 5.1 Boot 配置

| 设置             | EFI_13                     | EFI_15                     |
| ---------------- | -------------------------- | -------------------------- |
| LauncherOption   | `Disabled`               | `Full`                   |
| PickerMode       | `External`               | `External`               |
| PickerVariant    | `Acidanthera\GoldenGate` | `Acidanthera\GoldenGate` |
| Timeout          | `999`                    | `999`                    |
| PickerAttributes | `1`                      | `17`                     |

**LauncherOption 差异**:

- `Disabled`: OpenCore 不写入 NVRAM 启动项
- `Full`: OpenCore 在 UEFI 中写入最高优先级引导项，抵抗 Windows 更新覆盖

### 5.2 Tools 对比

| 工具                       | EFI_13      | EFI_15      |
| -------------------------- | ----------- | ----------- |
| CleanNvram                 | ✅ Enabled  | ✅ Enabled  |
| VerifyMsrE2                | ✅ Disabled | ✅ Disabled |
| **BootKicker.efi**   | ❌          | ✅ Enabled  |
| **ChipTune.efi**     | ❌          | ✅ Enabled  |
| **ControlMsrE2.efi** | ❌          | ✅ Enabled  |
| **CsrUtil.efi**      | ❌          | ✅ Enabled  |
| **GopStop.efi**      | ❌          | ✅ Enabled  |
| **KeyTester.efi**    | ❌          | ✅ Enabled  |
| **MmapDump.efi**     | ❌          | ✅ Enabled  |
| **OpenControl.efi**  | ❌          | ✅ Enabled  |
| **OpenShell.efi**    | ❌          | ✅ Enabled  |
| **ResetSystem.efi**  | ❌          | ✅ Enabled  |
| **RtcRw.efi**        | ❌          | ✅ Enabled  |
| **TpmInfo.efi**      | ❌          | ✅ Enabled  |

**EFI_15 工具优势**: 提供更多调试和维护工具，便于问题排查。

---

## 六、PlatformInfo 对比

| 设置               | EFI_13                                   | EFI_15                                   |
| ------------------ | ---------------------------------------- | ---------------------------------------- |
| SystemProductName  | `MacBookPro14,1`                       | `MacBookPro14,3`                       |
| MLB                | `C02824404GUHWVPCB`                    | `C02712902QXJ1JH1M`                    |
| SystemSerialNumber | `C02WVUYHHV29`                         | `C02TGBZ3HTD5`                         |
| SystemUUID         | `38704299-09E3-46EB-8635-82C4C422B9AF` | `08ABDD0B-0876-4649-9C05-954A37DF098C` |
| ROM                | `nPOHclh6`                             | `oChAWYTF`                             |
| ProcessorType      | `0`                                    | `1537`                                 |
| SpoofVendor        | `false`                                | `true`                                 |

**关键差异**:

- EFI_15 使用 `MacBookPro14,3` (i7-7700HQ 机型)，更接近实际硬件
- EFI_15 设置了 `ProcessorType = 1537` (Kaby Lake HQ 系列)
- EFI_15 启用了 `SpoofVendor = true`

---

## 七、Booter 配置对比

### 7.1 Quirks 差异

| 设置                | EFI_13    | EFI_15    |
| ------------------- | --------- | --------- |
| ResetLogoStatus     | `false` | `true`  |
| FixupAppleEfiImages | ❌ 无     | `true`  |
| ClearTaskSwitchBit  | ❌ 无     | `false` |

### 7.2 Patch 配置

**EFI_13**: 无 Booter Patch

**EFI_15**: 添加了 Board ID 检查跳过补丁

```xml
<Patch>
    <Comment>Skip Board ID check</Comment>
    <Find>PlatformSupport.plist</Find>
    <Replace>...........</Replace>
</Patch>
```

---

## 八、设备属性 (DeviceProperties) 对比

两者的设备属性基本一致，关键设置:

| 设备         | 关键属性            | 值                                |
| ------------ | ------------------- | --------------------------------- |
| Intel HD 630 | AAPL,ig-platform-id | `AAAbWQ==` (0x591b0000, 桌面版) |
| Intel HD 630 | hda-gfx             | `onboard-1`                     |
| ALC256 声卡  | layout-id           | `16` (AppleALC)                 |
| ALC256 声卡  | hda-gfx             | `onboard-1`                     |

---

## 九、Kernel Quirks 对比

| 设置                            | EFI_13    | EFI_15    |
| ------------------------------- | --------- | --------- |
| AppleXcpmCfgLock                | `true`  | `true`  |
| CustomSMBIOSGuid                | `true`  | `true`  |
| DisableIoMapper                 | `true`  | `true`  |
| PanicNoKextDump                 | `true`  | `true`  |
| PowerTimeoutKernelPanic         | `true`  | `true`  |
| XhciPortLimit                   | `true`  | `true`  |
| **SetApfsTrimTimeout**    | `0`     | `-1`    |
| **ForceSecureBootScheme** | `false` | `false` |
| **ExtendBTFeatureFlags**  | `false` | `false` |

### 9.1 新增 Quirks (EFI_15)

| 设置                | EFI_15   | 说明              |
| ------------------- | -------- | ----------------- |
| FixupAppleEfiImages | `true` | 处理苹果 EFI 图像 |

---

## 十、总结与建议

### 10.1 EFI_15 相对于 EFI_13 的改进

1. **更完整的 ACPI 补丁**: 新增 XOSI 相关补丁，更好支持新版 macOS
2. **更现代的 Kext 版本**: 大部分核心 Kext 已更新到最新版本
3. **更好的 USB 映射方案**: 采用 USBToolBox + UTBMap 替代静态 USBMap
4. **更完善的 WiFi 驱动**: itlwm 替代 AirportItlwm，避免版本绑定问题
5. **必要的 macOS 15 支持**: AMFIPass、RestrictEvents 等新 Kext
6. **更详细的 NVRAM 清理**: 确保配置修改能够生效
7. **更丰富的调试工具**: 14 个 UEFI 工具便于问题排查

### 10.2 EFI_13 的优势

1. **更简洁**: Kext 数量少，维护成本更低
2. **SIP 默认启用**: 安全性更好 (但会影响 OCLP 使用)
3. **使用 EnableJumpstart**: 无需 `apfs_aligned.efi`

### 10.3 建议

| 场景                        | 推荐配置                                                           |
| --------------------------- | ------------------------------------------------------------------ |
| **macOS 13 主力使用** | 使用 EFI_13，定期更新 Kext                                         |
| **macOS 15 主力使用** | 使用 EFI_15，配置已优化                                            |
| **多系统共存**        | 建议使用 EFI_15 作为统一引导，修改 config.plist 添加 macOS 13 条目 |
| **同步两个配置**      | 使用 OCAuxiliaryTools 进行合并，用 `ocvalidate` 验证             |

### 10.4 跨系统引导注意事项

如果需要在一个 EFI 中引导 macOS 13 和 macOS 15:

1. **Kext 兼容性**: 优先使用 EFI_15 的 Kext 集合
2. **SIP 配置**: 需要关闭 SIP 以支持 OCLP 功能
3. **NVRAM Delete**: 必须清理 `boot-args` 和 `csr-active-config`
4. **ProtocolOverrides**: 确保 `FirmwareVolume = true`

---

## 附录：快速参考表

### A. Kext 版本差异总览

| Kext                   | EFI_13 | EFI_15 |
| ---------------------- | ------ | ------ |
| Lilu                   | 1.6.6  | 1.7.3  |
| VirtualSMC             | 1.3.2  | 1.3.8  |
| WhateverGreen          | 1.6.5  | 1.7.1  |
| AppleALC               | 1.8.3  | 1.9.8  |
| VoodooI2C              | 2.8    | 2.9.1  |
| VoodooPS2              | 2.3.5  | 2.3.8  |
| RealtekRTL8111         | 2.4.2  | 3.0.4  |
| BlueToolFixup          | 2.6.7  | 2.7.2  |
| IntelBluetoothFirmware | 2.2.0  | 2.5.0  |

### B. 启动参数对比

```
EFI_13:
  -v

EFI_15:
  keepsyms=1 debug=0x100 -v -igfxblt -vi2c-force-polling revpatch=sbvmm -wegnoegpu ipc_control_port_options=0 -amfipassbeta -ibtcompatbeta
```

### C. 重要文件路径

| 配置类型     | EFI_13                         | EFI_15                         |
| ------------ | ------------------------------ | ------------------------------ |
| config.plist | `EFI_13/EFI/OC/config.plist` | `EFI_15/EFI/OC/config.plist` |
| ACPI         | `EFI_13/EFI/OC/ACPI/`        | `EFI_15/EFI/OC/ACPI/`        |
| Kexts        | `EFI_13/EFI/OC/Kexts/`       | `EFI_15/EFI/OC/Kexts/`       |
| Drivers      | `EFI_13/EFI/OC/Drivers/`     | `EFI_15/EFI/OC/Drivers/`     |

---

**报告结束**
