# 16GB U 盘黑苹果完整安装指南

**更新日期**：2026-03-29  
**OpenCore 版本**：1.0.7  
**macOS 版本**：Tahoe 26 / Big Sur 11.x

---

## 一、准备工作

### 硬件要求
| 物品 | 要求 |
|------|------|
| **U 盘** | 16GB 及以上 |
| **电脑** | 另一台 Mac（用于下载和制作） |
| **网络** | 约 12-14GB 下载流量 |
| **时间** | 约 1-2 小时 |

### 软件准备
| 工具 | 用途 | 下载链接 |
|------|------|---------|
| **Munki 脚本** | 下载 macOS 安装器 | https://github.com/munki/macadmin-scripts |
| **OpenCore 1.0.7** | 引导加载器 | https://github.com/acidanthera/OpenCorePkg/releases |
| **MountEFI** | 挂载 EFI 分区 | https://github.com/corpnewt/MountEFI |
| **ProperTree** | 编辑 config.plist | https://github.com/corpnewt/ProperTree |

---

## 二、U 盘分区结构

使用 `createinstallmedia` 命令后，U 盘会自动分成**两个分区**：

| 分区 | 名称 | 格式 | 用途 | 大小 |
|------|------|------|------|------|
| **主分区** | `MyVolume` | Mac OS 扩展（HFS+） | macOS 安装器 | 约 12-14GB |
| **EFI 分区** | `EFI` | FAT32 | 引导分区（OpenCore） | 约 200MB |

**磁盘工具显示：**
```
disk2 (16GB USB)
├─ disk2s1 (EFI, 200MB)        ← 隐藏分区，需挂载
└─ disk2s2 (MyVolume, 14GB)    ← 安装器分区
```

---

## 三、完整制作流程

### 步骤 1：安装依赖

```bash
# 安装 xattr 模块
pip3 install xattr
```

---

### 步骤 2：下载 Munki 脚本

```bash
# 创建目录
mkdir -p ~/macOS-installer && cd ~/macOS-installer

# 下载脚本
curl https://raw.githubusercontent.com/munki/macadmin-scripts/main/installinstallmacos.py > installinstallmacos.py
```

---

### 步骤 3：下载 macOS 安装器

```bash
# 运行脚本（需要管理员密码）
sudo python3 installinstallmacos.py
```

**脚本会列出所有可用版本：**
```
#   ProductID            Version    Build      Post Date  Title
1  000-xxxxx            26.0       25A354     2026-03-20  macOS Tahoe 26
2  001-xxxxx            15.7.2     15A123     2026-03-15  macOS Sequoia 15
3  002-xxxxx            14.7.5     14A123     2026-03-10  macOS Sonoma 14
4  003-xxxxx            13.7.5     13A123     2026-03-05  macOS Ventura 13
5  004-xxxxx            12.7.5     12A123     2026-03-01  macOS Monterey 12
6  005-xxxxx            11.7.10    11A123     2026-02-25  macOS Big Sur 11

Choose a product to download (1-6): 1
```

**输入对应数字选择版本，等待下载完成（约 30-60 分钟）**

---

### 步骤 4：格式化 U 盘

**磁盘工具操作：**
1. 打开「磁盘工具」（应用程序 → 实用工具）
2. 选择 U 盘（显示为整个磁盘，不是分区）
3. 点击「抹掉」
4. 设置：
   - **名称**：`MyVolume`
   - **格式**：`Mac OS 扩展（日志式）`
   - **方案**：`GUID 分区图`
5. 点击「抹掉」

**这会创建两个分区：**
- `MyVolume` - 主分区
- `EFI` - 隐藏的引导分区（200MB FAT32）

---

### 步骤 5：创建可启动 U 盘

```bash
# 根据下载的 macOS 版本选择对应命令：

# macOS Tahoe 26
sudo /Applications/Install\ macOS\ Tahoe.app/Contents/Resources/createinstallmedia --volume /Volumes/MyVolume

# macOS Sequoia 15
sudo /Applications/Install\ macOS\ Sequoia.app/Contents/Resources/createinstallmedia --volume /Volumes/MyVolume

# macOS Sonoma 14
sudo /Applications/Install\ macOS\ Sonoma.app/Contents/Resources/createinstallmedia --volume /Volumes/MyVolume

# macOS Ventura 13
sudo /Applications/Install\ macOS\ Ventura.app/Contents/Resources/createinstallmedia --volume /Volumes/MyVolume

# macOS Monterey 12
sudo /Applications/Install\ macOS\ Monterey.app/Contents/Resources/createinstallmedia --volume /Volumes/MyVolume

# macOS Big Sur 11
sudo /Applications/Install\ macOS\ Big\ Sur.app/Contents/Resources/createinstallmedia --volume /Volumes/MyVolume
```

**等待完成（约 10-30 分钟）**

---

### 步骤 6：下载 OpenCore 1.0.7

```bash
# 下载最新 release
# https://github.com/acidanthera/OpenCorePkg/releases/download/1.0.7/OpenCore-1.0.7-RELEASE.zip

# 或使用命令行
curl -L https://github.com/acidanthera/OpenCorePkg/releases/download/1.0.7/OpenCore-1.0.7-RELEASE.zip -o ~/Downloads/OpenCore-1.0.7-RELEASE.zip

# 解压
unzip ~/Downloads/OpenCore-1.0.7-RELEASE.zip -d ~/Downloads/
```

---

### 步骤 7：挂载 EFI 分区

**方法 A：使用 MountEFI 工具**
```bash
# 下载 MountEFI
# https://github.com/corpnewt/MountEFI

# 运行 MountEFI，选择 U 盘的 EFI 分区
```

**方法 B：使用磁盘工具**
1. 打开「磁盘工具」
2. 点击「显示」→「显示所有设备」
3. 找到 U 盘的 EFI 分区
4. 点击「挂载」

---

### 步骤 8：写入 OpenCore 引导

1. **复制 EFI 文件夹**
   ```
   从 OpenCore-1.0.7-RELEASE/X64/EFI/OC/ 
   复制到 U 盘 EFI 分区/EFI/OC/
   ```

2. **复制 BOOTx64.efi**
   ```
   从 OpenCore-1.0.7-RELEASE/X64/BOOT/BOOTx64.efi
   复制到 U 盘 EFI 分区/EFI/BOOT/BOOTx64.efi
   ```

3. **目录结构**
   ```
   U 盘 EFI 分区/
   ├── BOOT/
   │   └── BOOTx64.efi
   └── OC/
       ├── ACPI/
       ├── Drivers/
       │   ├── OpenCanopy.efi
       │   ├── HfsPlus.efi
       │   └── ...
       ├── Kexts/
       │   ├── Lilu.kext
       │   ├── VirtualSMC.kext
       │   ├── WhateverGreen.kext
       │   └── ...
       ├── Resources/
       ├── Tools/
       └── config.plist
   ```

---

### 步骤 9：配置 config.plist

**使用 ProperTree 或 OCAT 编辑：**

1. **PlatformInfo** - 设置 SMBIOS 型号
2. **Booter** - 配置 Quirks
3. **Kernel** - 添加 Kexts 和 Patch
4. **UEFI** - 配置 Drivers 和 APFS
5. **Misc** - 配置启动项

**常用 Kexts：**
| Kext | 用途 |
|------|------|
| Lilu.kext | 内核补丁框架 |
| VirtualSMC.kext | SMC 模拟 |
| WhateverGreen.kext | 显卡驱动 |
| AppleALC.kext | 声卡驱动 |
| IntelMausi.kext | Intel 网卡 |
| AirportItlwm.kext | Intel 无线网卡 |

---

### 步骤 10：测试启动

1. 重启电脑
2. 按住 `Option` 键（Intel Mac）或长按电源键（Apple Silicon）
3. 选择 `EFI Boot` 或 `OpenCore`
4. 在 OpenCore 菜单中选择 `Install macOS Tahoe`

---

## 四、启动流程

```
开机 → 按住 Option 键 → 选择 EFI 分区（显示为 "EFI Boot"）
     → OpenCore 菜单 → 选择 "Install macOS Tahoe"
     → 进入安装器
```

---

## 五、重要注意事项

| 问题 | 说明 |
|------|------|
| **macOS 11.3+ USB 端口** | 需使用 USBToolBox 映射 USB 端口 |
| **XhciPortLimit** | macOS 11.3+ 已失效，需禁用或映射 USB |
| **EFI 分区隐藏** | 默认隐藏，需用 MountEFI 或磁盘工具挂载 |
| **U 盘名称** | 必须为 `MyVolume`，否则 createinstallmedia 失败 |
| **16GB 容量** | macOS Tahoe 约 12-14GB，剩余空间给 EFI 分区 |
| **8GB U 盘限制** | 只能装 Big Sur 11.x 及更早版本 |

---

## 六、命令速查表

| 操作 | 命令 |
|------|------|
| **安装依赖** | `pip3 install xattr` |
| **下载脚本** | `curl https://raw.githubusercontent.com/munki/macadmin-scripts/main/installinstallmacos.py > installinstallmacos.py` |
| **运行脚本** | `sudo python3 installinstallmacos.py` |
| **创建 U 盘** | `sudo /Applications/Install\ macOS\ [版本].app/Contents/Resources/createinstallmedia --volume /Volumes/MyVolume` |
| **下载 OpenCore** | `curl -L https://github.com/acidanthera/OpenCorePkg/releases/download/1.0.7/OpenCore-1.0.7-RELEASE.zip -o ~/Downloads/OpenCore-1.0.7-RELEASE.zip` |

---

## 七、官方资源链接

| 资源 | 链接 |
|------|------|
| **OpenCore 官方指南** | https://dortania.github.io/OpenCore-Install-Guide/ |
| **OpenCore 下载** | https://github.com/acidanthera/OpenCorePkg/releases |
| **Munki 脚本** | https://github.com/munki/macadmin-scripts |
| **MountEFI** | https://github.com/corpnewt/MountEFI |
| **ProperTree** | https://github.com/corpnewt/ProperTree |
| **USBToolBox** | https://usbtoolbox.github.io/ |
| **Apple 官方支持** | https://support.apple.com/en-us/HT201372 |

---

## 八、故障排查

### 找不到启动盘
| 原因 | 解决方法 |
|------|---------|
| OpenCore 版本过旧 | 升级到 1.0.7 |
| 缺少 HfsPlus.efi 驱动 | 添加 HfsPlus.efi 到 Drivers 目录 |
| ScanPolicy 设置错误 | 设置 ScanPolicy = 0 |
| MinDate/MinVersion 限制 | 设置为 -1 |

### 无限重启（Boot Loop）
| 原因 | 解决方法 |
|------|---------|
| XhciPortLimit 失效 | 禁用 XhciPortLimit 或使用 USBToolBox 映射 |
| USB 端口未映射 | 使用 USBToolBox 映射 USB 端口 |
| config.plist 配置错误 | 检查 Booter Quirks 配置 |

---

**文档生成时间**：2026-03-29  
**适用系统**：macOS Tahoe 26 / Sequoia 15 / Sonoma 14 / Ventura 13 / Monterey 12 / Big Sur 11  
**OpenCore 版本**：1.0.7
