# JLink V9

JLink V9 调试器项目，包含固件、原理图、许可证和相关文档。

## 项目结构

```
jlink_v9/
├── sch/                    # 原理图文件
│   ├── JlinkV9_mini.pdf
│   └── lm358.png
├── doc/                    # 文档和规格书
│   ├── STM32F20x.svd
│   ├── stm32f205rc.pdf
│   ├── lm358.pdf
│   ├── sn74lvc2t45.pdf
│   ├── sn74alvc164245.pdf
│   ├── C330092_静电和浪涌保护(TVS-ESD)_PRTR5V0U2X_规格书_元基静电放电(ESD)保护器件规格书.PDF
│   └── 自制固件主程序.rar
├── Firmwares/              # 固件文件
│   └── JLink_V9_20210507162612_0x08000000.bin
├── bootloader/             # 引导加载程序
│   ├── JLink_v9_bootloader_20121012_no_rdp_0x08000000.bin
│   ├── JLink_v9_bootloader_20121012_0x08000000.bin
│   └── RDP_Analysis/       # RDP分析和修改
│       ├── RDP_Analysis_and_Modification.md
│       ├── bootloader_disassembly.txt
│       └── 反汇编相关文件
├── app/                    # 应用程序
│   ├── JLink_V9_app_20150220092019_0x08010000.bin
│   ├── JLink_V9_app_20170601192241_0x08010000.bin
│   ├── JLink_V9_app_20170719161145_0x08010000.bin
│   └── JLink_V9_app_20210507162612_0x08010000.bin
└── license/                # 许可证相关
    ├── JLink_V9_version_0x0800C000.bin
    ├── JLink_V9_license_0x08008000.bin
    ├── jlink-v9激活.txt
    └── 添加license/
        ├── 方法1/
        │   └── Segger_J-Link_keygen.exe
        └── 方法2/
            ├── JLink.exe
            ├── JLinkARM.dll
            ├── run.txt
            └── run.bat
```

## 硬件信息

- 主控芯片: STM32F205RC
- 运算放大器: LM358
- 电平转换: SN74LVC2T45, SN74ALVC164245
- ESD保护: PRTR5V0U2X

## 固件说明

### Bootloader
- 原始版本: 2012-10-12
- 无RDP保护版本: 可用于分析和修改

### 应用程序
- 2015-02-20 版本
- 2017-06-01 版本
- 2017-07-19 版本
- 2021-05-07 版本 (最新)

## 许可证激活

项目提供两种许可证激活方法：

### 方法1
使用 `Segger_J-Link_keygen.exe` 生成许可证密钥。

### 方法2
运行 `run.bat` 脚本，使用 `JLink.exe` 和 `JLinkARM.dll` 进行激活。

详细说明请参考 `jlink-v9激活.txt` 文件。

## 注意事项

- 使用本项目的固件和许可证可能涉及法律风险，请确保在合法范围内使用
- 修改和烧录固件前请备份原始数据
- RDP (Read Out Protection) 相关操作可能导致设备不可逆损坏

## 相关文档

- [RDP分析与修改说明](bootloader/RDP_Analysis/RDP_Analysis_and_Modification.md)
- 原理图: `sch/JlinkV9_mini.pdf`
- 芯片规格书: `doc/` 目录下的PDF文件

## 许可声明

本项目仅供学习和研究使用。
