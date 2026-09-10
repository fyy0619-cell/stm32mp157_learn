# 02 · Kernel 启动层 —— VFS 挂载与 rootfs

> 本章覆盖内核起来后**挂载根文件系统失败**这一大类问题,以及若干"启动过程中的红字警告到底要不要理"的判断题。
> 阅读时长:约 20 分钟

---

## 坑 2-1:`VFS: Cannot open root device "nfs" or unknown-block(2,0): error -6`

### 现象

内核完整启动,但最后 Kernel panic:

```
VFS: Unable to mount root fs via NFS, trying floppy.
VFS: Cannot open root device "nfs" or unknown-block(2,0): error -6
Please append a correct "root=" boot option; here are the available partitions:
0100           65536 ram0
 (driver?)
0101           65536 ram1
 (driver?)
...
0109           65536 ram9
 (driver?)
Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(2,0)
```

且 bootargs 明确写了 `root=/dev/nfs nfsroot=192.168.5.10:/home/fu/nfs/rootfs,proto=tcp`。

### 定位思路(这是本笔记最有价值的一段)

初学者第一反应:"NFS 服务挂了 / bootargs 写错了"。**这两个方向都是坑,先别急着改。**

**正确的定位路径**:

**Step 1** —— 看错误码。`error -6` = `ENXIO` = "No such device or address"。**字面意思:根本没这个设备**。不是"找不到 IP",不是"权限不对",是"设备不存在"。

**Step 2** —— 看"available partitions"列表。只有 `ram0..ram15`,**没有任何真实块设备,也没有网络设备**。这说明内核根本没识别到网卡。

**Step 3** —— 反向搜索日志里跟以太网相关的字符串。有的就一句 `libphy: Fixed MDIO Bus: probed`(通用框架),**没有 `stmmac`、`dwmac`、`eth0: registered` 这些真正的驱动加载行**。

**Step 4** —— 找 `IP-Config: Complete:` 这行。**没找到**。正常情况下 kernel 用 `ip=` 参数配好网卡后必然打这行,没打说明网卡在 kernel 阶段根本没起来。

**结论**:根因不是 NFS,不是 bootargs,是**内核里没有以太网驱动 / DTS 里没使能网卡**。属于 Stage 4 内部的 driver probe 层。

### 根因

两种可能:

1. **驱动没编进内核**:`.config` 里 `CONFIG_STMMAC_ETH=n` 或 `=m`(模块不行,rootfs 都没挂上哪来的 modprobe)
2. **DTS 没使能**:SoC 级 dtsi 里 `ethernet0` 默认 `status = "disabled"`,板级 dtsi 没 override 成 `okay`

本项目实际命中的是**第 2 种**(见 [03 章](./03-设备树驱动层-DTS移植与stmmac.md))。

### 解决

先验证 kernel config:
```bash
grep -E "STMMAC|DWMAC_STM32" .config
# 期望看到:
# CONFIG_STMMAC_ETH=y
# CONFIG_STMMAC_PLATFORM=y
# CONFIG_DWMAC_STM32=y
```

再验证 DTS:
```bash
grep -A 5 "&ethernet0" arch/arm/boot/dts/stm32mp157d-atk.dtsi
# 期望看到:
# &ethernet0 {
#     status = "okay";
#     ...
# };
```

**任意一个没满足,就补上再重编内核和 dtb**。

### 背后原理

Linux `errno` 值对应的语义要熟:

| errno | 名字 | 常见场景 |
|---|---|---|
| -2 | ENOENT | 文件/路径不存在 |
| -5 | EIO | 硬件 I/O 错误 |
| **-6** | **ENXIO** | **设备不存在** |
| -12 | ENOMEM | 内存不足 |
| -13 | EACCES | 权限拒绝 |
| -17 | EEXIST | 已存在 |
| -19 | ENODEV | 无此设备(和 -6 相近,但语义不同) |
| -22 | EINVAL | 参数无效 |
| -110 | ETIMEDOUT | 超时 |

**-6 和 -19 的区别**:-6 通常表示"路径/名字对了但设备没起来",-19 表示"根本没这个设备类型"。VFS 报 -6 就是"你说 root=nfs,但我没有网络这个类型的 root 设备可用"。

### 岗位映射

**这是本仓库最能体现 BSP 能力的一个坑。** 面试话术:

> "有次 NFS rootfs 挂不上,内核 panic。表面看是 NFS 的问题,但我没直接去改 NFS 配置,而是先看了三个东西:一是 errno -6 是 ENXIO,意思是'没这个设备';二是 available partitions 列表里只有 ram*,没有网卡;三是 grep 整段日志找 stmmac 相关行,一行都没有,而且 IP-Config: Complete 也没打。三个证据指向同一个结论:根本原因是内核里没有以太网设备。后来定位到是板级 dtsi 缺 ethernet0 节点。这个经历让我理解到,rootfs 挂不上是**症状**,rootfs 依赖的驱动没起来才是**根因**——嵌入式调试永远要往前一层一层追。"

---

## 坑 2-2:`stpmic1 0-0033: Unable to read PMIC version`

### 现象

内核日志里的红字:

```
stm32f7-i2c 5c002000.i2c: doesn't use DMA
stpmic1 0-0033: Unable to read PMIC version
stm32f7-i2c 5c002000.i2c: STM32F7 I2C-0 bus adapter
```

### 定位思路

**先判断:这是致命错误还是可忽略警告?** 看接下来内核有没有继续正常初始化其他外设。如果 MMC、USB、以太网(如果配了)都能正常出现,说明这个 PMIC 报错不阻塞整体启动。

`stpmic1 0-0033` 的含义:
- `0-0033`:i2c bus 0 上的地址 0x33
- `stpmic1`:ST 的电源管理芯片(STPMIC1)驱动

驱动想通过 I2C 读 PMIC 版本寄存器,读失败。

### 根因(按可能性排)

1. **DTS 里 PMIC 节点在 i2c4,但 ATK 板子可能实际没这颗芯片或者用了别的电源方案**(比如分立 DCDC)
2. **I2C 拉高电阻缺失** / **PMIC 未上电**
3. **PMIC 型号不匹配**:STPMIC1 有多个 revision,驱动认不出

对 ATK STM32MP157-ED1 板子,通常是**情况 1**——ATK 板子 dtsi 里配了 STPMIC1 节点(继承自 ST 官方 ED1 参考设计),但实际板载电源方案可能不同。

### 解决

**如果系统能正常启动(能进 shell),就忽略。** 这个 PMIC 只影响一些高级电源管理功能(动态调压、待机深睡眠),不影响基本工作。

如果要彻底消错:
1. 在 dtsi 里把 stpmic 节点删掉或 `status = "disabled"`
2. 手动配好其他电源节点

### 背后原理

STM32MP157 的电源域比较复杂:

```
主 SoC 供电域 (Cortex-A7)   ← 1.1V, buck1 供
DDR 供电域                   ← 1.35V, buck2 供
主 3V3                       ← buck3
外设 3V3                     ← buck4
模拟电源 (VDDA)              ← ldo1
USB 3V3                      ← ldo4
SD 卡 2.9V                   ← ldo5
```

如果用 STPMIC1 方案,内核通过驱动统一管理这些电压;如果用分立 DCDC,这些电压由硬件默认值固定。两种方案对普通工作没差别,但**低功耗模式必须走 PMIC**。

### 岗位映射

**锻炼的能力:识别"哪些报错必须修、哪些可以忽略"。**

面试问"启动日志有红字怎么办",初级答"全部修掉",高级答"先分类,看是否阻塞主功能。STPMIC1 版本读失败是典型的可忽略红字,只影响高级电源管理"。这个判断能省几个小时的无效排查。

---

## 坑 2-3:`ALSA device list: No soundcards found`

### 现象

```
ALSA device list:
  No soundcards found.
```

### 定位思路

`No soundcards found` 本质是**没启用音频节点**。判断要不要修的标准很简单:**你这块板子有没有用到音频功能?**

### 根因

`.config` 或 DTS 里没启用音频 codec 相关节点。ATK STM32MP157 默认 dts 里 sound 节点是 disable 的。

### 解决

**不用音频就忽略**。要用的话:
1. dts 里加 sound 节点,指向具体 codec (WM8994 / SGTL5000 等)
2. `.config` 启用对应 codec 驱动
3. 重编 kernel

### 岗位映射

跟坑 2-2 同类——**识别可忽略的报错**。

---

## 坑 2-4:`RTC: Date/Time must be initialized`

### 现象

```
stm32_rtc 5c004000.rtc: registered as rtc0
stm32_rtc 5c004000.rtc: Date/Time must be initialized
stm32_rtc 5c004000.rtc: registered rev:1.2
...
stm32_rtc 5c004000.rtc: setting system clock to 2000-01-08T04:49:55 UTC (947306995)
```

### 根因

RTC 芯片里没有有效时间。原因通常是:
- 首次上电
- 掉电时间过长,后备电池没接或没电
- 板子没设计后备电池

### 解决

**用户态处理**,不是驱动/内核问题:
```bash
date -s "2026-09-10 20:00:00"
hwclock -w                       # 写回 RTC
```

联网后可以自动 NTP:
```bash
timedatectl set-ntp true
```

### 岗位映射

**理解 RTC 依赖后备电池、系统时间与硬件时间分离的概念。**

面试问"系统时间怎么保存",能答出:
- 运行时时间在 kernel 的 jiffies 里
- 系统时间源自 RTC 硬件(每次开机从 RTC 读)
- RTC 靠板载后备电池(纽扣电池)保持
- 联网设备用 NTP 校准

这四层关系答出来就足够体现基本功。

---

## 本章相关的面试问题

1. **看到 `error -6` 你会怎么解读?为什么不能直接改 NFS 配置?**
2. **内核 panic on VFS,可用块设备只有 ram*,说明什么?下一步查什么?**
3. **`IP-Config: Complete` 这行日志的意义?没出现说明什么?**
4. **启动日志里的红字/警告,你怎么判断哪些必须修、哪些可以忽略?**
5. **STPMIC1 读版本失败,你会怎么排查?怎么判断优先级?**
6. **RTC 掉电时间没了,应该在哪一层解决?**

---

## 下一章

VFS 挂不上根因往下追,80% 会追到 DTS + 驱动的问题 → [03-设备树驱动层-DTS移植与stmmac.md](./03-设备树驱动层-DTS移植与stmmac.md)
