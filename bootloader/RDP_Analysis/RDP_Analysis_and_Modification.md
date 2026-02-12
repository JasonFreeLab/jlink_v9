# J-Link V9 Bootloader RDP保护分析与修改

## 目录
1. [背景介绍](#背景介绍)
2. [RDP保护机制](#rdp保护机制)
3. [代码分析](#代码分析)
4. [修改方案](#修改方案)
5. [修改实施](#修改实施)
6. [验证方法](#验证方法)
7. [附录](#附录)
8. [结论](#结论)

---

## 背景介绍

J-Link V9 bootloader固件在启动时会设置STM32F205的RDP（Read Out Protection）保护，导致无法通过调试接口读取Flash内容。本文档详细分析了RDP设置代码，并提供了禁用RDP保护的方法。

### 问题现象

烧录原始固件后，出现以下问题：
```
> dump_image upload_firmware.bin 0x08000000 262144
Failed to read memory at 0x08000004

> flash info 0
Device Security Bit Set
stm32f2x user_options 0xEC
```

---

## RDP保护机制

### STM32F2 RDP级别

| 级别 | RDP位值 | 保护状态 | 描述 |
|------|---------|----------|------|
| Level 0 | 0xAA | 无保护 | 可以完全访问Flash |
| Level 1 | 0x55 | 读保护 | 禁止调试读取Flash，但允许执行 |
| Level 2 | 0xCC (永久) | 永久保护 | 禁止调试和JTAG/SWD，不可逆 |

### 关键寄存器

| 寄存器 | 地址 | 描述 |
|--------|------|------|
| FLASH_OPTKEYR | 0x40023C08 | Flash选项密钥寄存器 |
| FLASH_OPTCR | 0x40023C14 | Flash选项控制寄存器 |
| 选项字节 | 0x1FFF7800 | 选项字节存储地址 |

### 选项密钥

要修改选项字节，必须先写入密钥：
- KEY1: 0x0819 2A3B
- KEY2: 0x4C5D 6E7F

---

## 代码分析

### 反汇编bin文件
```bash
arm-none-eabi-objdump -D -b binary -m arm -M force-thumb --adjust-vma=0x08000000 JLink_v9_bootloader_20121012_0x08000000.bin > bootloader_disassembly.txt
```

### 关键代码位置

在 `bootloader_disassembly.txt` 的第6295-6319行找到RDP设置代码：

```assembly
8003c56: f3ef 8010  mrs r0, PRIMASK           ; 保存中断状态
8003c5a: b672       cpsid i                   ; 禁用中断
8003c5c: 4915       ldr r1, [pc, #84]        ; 加载FLASH_ACR地址(0x40023C00)
8003c5e: 68ca       ldr r2, [r1, #12]        ; 读取FLASH_OPTCR (0x40023C14)
8003c60: f3c2 2307  ubfx r3, r2, #8, #8      ; 提取bit 8-15
8003c64: 2baa       cmp r3, #170 @ 0xaa       ; 比较是否为0xAA
8003c66: d112       bne.n 0x8003c8e          ; 如果不是0xAA，跳过设置
8003c68: 07d3       lsls r3, r2, #31         ; 检查bit31
8003c6a: d505       bpl.n 0x8003c78          ; 如果bit31为0，跳过设置
8003c6c: 4b12       ldr r3, [pc, #72]        ; 加载第一个选项密钥 (0x08192A3B)
8003c6e: 600b       str r3, [r1, #0]         ; 写入FLASH_OPTKEYR (0x40023C08)
8003c70: 4b12       ldr r3, [pc, #72]        ; 加载第二个选项密钥 (0x4C5D6E7F)
8003c72: 600b       str r3, [r1, #0]         ; 写入FLASH_OPTKEYR
8003c74: 0852       lsrs r2, r2, #1          ; 清除RDP位
8003c76: 0052       lsls r2, r2, #1          ; 恢复其他位
8003c78: f442 427f  orr.w r2, r2, #65280   @ 0xff00 ; 设置其他选项位
8003c7c: 60ca       str r2, [r1, #12]        ; 写入FLASH_OPTCR (设置选项字节)
8003c7e: f042 0202  orr.w r2, r2, #2        ; 设置OPTSTRT位
8003c82: 60ca       str r2, [r1, #12]        ; 再次写入FLASH_OPTCR (触发编程)
8003c84: 684a       ldr r2, [r1, #4]        ; 读取FLASH_SR
8003c86: f3c2 4200  ubfx r2, r2, #16, #1    ; 检查BSY位
8003c8a: 2a00       cmp r2, #0               ; 等待编程完成
8003c8c: d1fa       bne.n 0x8003c84          ; 循环等待
8003c8e: f380 8810  msr PRIMASK, r0          ; 恢复中断状态
8003c92: 4770       bx lr                     ; 返回
```

### 代码逻辑分析

1. **条件检查** (8003c5e-8003c6a):
   - 读取当前FLASH_OPTCR值
   - 检查bit 8-15是否为0xAA（RDP Level 0）
   - 检查bit31是否为1（某个标志位）
   - 只有满足条件才执行RDP设置

2. **解锁选项字节** (8003c6c-8003c72):
   - 写入选项密钥到FLASH_OPTKEYR
   - KEY1: 0x0819 2A3B
   - KEY2: 0x4C5D 6E7F

3. **设置选项字节** (8003c74-8003c82):
   - 清除RDP位（bit 0-1）
   - 设置其他选项位（0xFF00）
   - 写入FLASH_OPTCR寄存器
   - 设置OPTSTRT位触发编程

4. **等待完成** (8003c84-8003c8c):
   - 检查FLASH_SR的BSY位
   - 等待编程完成

### 关键指令

| 地址 | 指令 | 描述 |
|------|------|------|
| 0x8003C7C | `60ca` | STR R2, [R1, #12] - 写入FLASH_OPTCR |
| 0x8003C82 | `60ca` | STR R2, [R1, #12] - 触发选项字节编程 |

这两条指令是设置RDP保护的关键，需要将其替换为NOP。

---

## 修改方案

### 修改目标

禁用RDP保护设置功能，使bootloader不再自动启用读保护。

### 修改方法

将两条STR指令替换为NOP（空操作）指令：

1. **地址0x8003C7C**:
   - 原始: `ca 60` (STR R2, [R1, #12])
   - 修改后: `bf 00` (NOP)

2. **地址0x8003C82**:
   - 原始: `ca 60` (STR R2, [R1, #12])
   - 修改后: `bf 00` (NOP)

### 修改原理

- NOP指令不执行任何操作，只是占用一个指令周期
- 替换STR指令后，不会写入FLASH_OPTCR寄存器
- RDP保护不会被启用，Flash保持可读状态

### 对固件功能的影响

| 功能 | 影响 | 说明 |
|------|------|------|
| Bootloader启动 | 无影响 | Bootloader正常启动和运行 |
| 固件升级 | 无影响 | 升级功能正常工作 |
| RDP保护 | 禁用 | 不再自动启用读保护 |
| Flash读取 | 正常 | 可以通过调试接口读取Flash |
| Flash写入 | 正常 | 可以正常编程Flash |

---

## 修改实施

### 原始文件

```h
00003c70  12 4b 0b 60 52 08 52 00  42 f4 7f 42 ca 60 42 f0  |.K.`R.R.B..B.`B.|
00003c80  02 02 ca 60 4a 68 c2 f3  00 42 00 2a fa d1 80 f3  |...`Jh...B.*....|
```

### 修改结果

```h
00003c70  12 4b 0b 60 52 08 52 00   42 f4 7f 42 bf 00 42 f0  |.K.`R.R.B..B..B.|
00003c80  02 02 bf 00 4a 68 c2 f3   00 42 00 2a fa d1 80 f3  |....Jh...B.*....|
```

可以看到：
- 地址0x00003c7c-0x00003c7d: `bf 00` (NOP) ✅
- 地址0x00003c82-0x00003c83: `bf 00` (NOP) ✅

---

## 验证方法

### 1. 烧录固件

使用OpenOCD烧录修改后的固件：

```bash
openocd -f interface/cmsis-dap.cfg -f target/stm32f2x.cfg
```

在telnet命令行中：

```tcl
init
reset halt
flash write_image erase JLink_v9_bootloader_20121012_no_rdp_0x08000000.bin 0x08000000
reset run
```

### 2. 验证选项字节

```tcl
stm32f2x options_read 0
```

**预期结果**：
```
stm32f2x user_options 0xAA  (或非0xEC的值)
```

**不应该看到**：
```
Device Security Bit Set
stm32f2x user_options 0xEC
```

### 3. 验证Flash读取

```tcl
dump_image upload_firmware.bin 0x08000000 262144
```

**预期结果**：
```
Successfully dumped 262144 bytes
```

**不应该看到**：
```
Failed to read memory at 0x08000004
```

### 4. 验证Bootloader功能

1. 检查Bootloader是否正常启动
2. 检查固件升级功能是否正常
3. 检查应用程序是否正常跳转和运行

---

## 附录

### A. 相关文件

| 文件名 | 描述 |
|--------|------|
| `JLink_v9_bootloader_20121012_0x08000000.bin` | 原始bootloader固件 |
| `JLink_v9_bootloader_20121012_0x08000000_no_rdp.bin` | 修改后的bootloader固件 |
| `bootloader_disassembly.txt` | 反汇编文件 |
| `JLink_v9_bootloader_20121012_0x08000000.bin.i64` | IDA pro 9.3 工程 |
| `JLink_v9_bootloader_20121012_0x08000000.bin.asm` | IDA反汇编文件 |
| `JLink_v9_bootloader_20121012_0x08000000.bin.c` | IDA反编译文件 |

### B. 参考文档

1. STM32F2xx Reference Manual - RM0033
2. STM32F205 Datasheet
3. ARM Cortex-M3 Technical Reference Manual
4. OpenOCD User Guide

### C. 修改记录

| 日期 | 版本 | 修改内容 | 作者 |
|------|------|----------|------|
| 2026-02-12 | 1.0 | 初始版本，完成RDP分析和修改 | JasonFreeLab |

---

## 结论

通过分析J-Link V9 bootloader的RDP设置代码，我们成功定位了设置读保护的关键指令，并提供了禁用RDP保护的修改方案。修改后的固件可以正常工作，但不会自动启用RDP保护，从而允许通过调试接口读取Flash内容。

**重要提醒**：此修改仅适用于开发和学习环境，在生产环境中使用需要仔细评估安全风险。

---

*文档版本：1.0*
*最后更新：2026-02-12*
