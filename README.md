# STM32MP157_learn —— 嵌入式 Linux 学习踩坑实录

> 目标读者：AI BSP / NPU 板级支持 / 嵌入式 Linux 驱动方向,准备 **2027 秋招** 的应届/在读同学
> 平台：正点原子 STM32MP157D-ATK 开发板 (Cortex-A7 双核 + Cortex-M4)
> 写作动机：把学 ATK STM32MP157 时撞的每一个坑,按 **BSP 工程师的分析框架** 复盘一遍。目标不是记流水账,而是把"环境噪音"翻译成"面试可讲的能力证据"。

---

## 一、为什么按"启动链分层"组织,而不是按时间线?

流水式笔记(先撞 A、后撞 B)对**当事人复盘**有用,对**面试官**没用。面试官关心的是:**"你遇到问题,是怎么分层定位的?"**

所以本仓库按嵌入式 Linux 从上电到用户空间的**启动链五层**组织,每一层对应一类问题域和一套排查方法论:

```
 ┌─────────────────────────────────────────────────┐
 │  用户空间层  bash / shell / 应用                 │ ← 登录、命令、误操作
 ├─────────────────────────────────────────────────┤
 │  根文件系统层  rootfs / NFS / initramfs          │ ← VFS 挂载、init 启动
 ├─────────────────────────────────────────────────┤
 │  内核层  vmlinux + drivers + DTS                │ ← probe、PHY、stmmac
 ├─────────────────────────────────────────────────┤
 │  Bootloader 层  U-Boot / TF-A                    │ ← bootm、bootargs、tftp
 ├─────────────────────────────────────────────────┤
 │  片上启动 ROM + PMIC 上电时序                    │ ← stpmic1、时钟树
 └─────────────────────────────────────────────────┘
                       ▲
                  硬件(SoC / DDR / PHY / eMMC / MMC)
```

**BSP 工程师的核心能力 = 在这五层之间自由切换,遇到现象能第一时间猜出属于哪一层、去哪层看日志、改哪层配置。** 这份笔记就是围绕这个能力训练的。

---

## 二、阅读路径

| 顺序 | 文档 | 你将学到 | 面试对应能力 | 估计阅读 |
|---|---|---|---|---|
| 0 | [00-引言-嵌入式Linux启动链与BSP工程师视角.md](./00-引言-嵌入式Linux启动链与BSP工程师视角.md) | 启动链五层拆解、BSP 工程师日常在哪几层作战 | 全链路认知 | 10 min |
| 1 | [01-Bootloader层-U-Boot命令语法与环境变量.md](./01-Bootloader层-U-Boot命令语法与环境变量.md) | bootm 参数、bootcmd 组装、TFTP 命令、ARP 排查 | Bootloader 熟悉度 | 20 min |
| 2 | [02-Kernel启动层-VFS挂载与rootfs.md](./02-Kernel启动层-VFS挂载与rootfs.md) | Kernel panic 定位、`error -6` 含义、IP-Config 是否 complete | 内核启动流程 | 20 min |
| 3 | [03-设备树驱动层-DTS移植与stmmac.md](./03-设备树驱动层-DTS移植与stmmac.md) | 板级 dts/dtsi 分工、status 属性、PHY 描述、driver ↔ DTS probe 关系 | DTS + 驱动移植 | 25 min |
| 4 | [04-网络传输层-TFTP与NFS调试方法论.md](./04-网络传输层-TFTP与NFS调试方法论.md) | 分层 ping、NFS v3/v4 版本坑、exports 权限、bootargs ip= 语法 | 网络排障 | 20 min |
| 5 | [05-交叉编译环境-Buildroot新老系统兼容性.md](./05-交叉编译环境-Buildroot新老系统兼容性.md) | glibc 2.34+ 破坏点(SIGSTKSZ / _STAT_VER)、老 buildroot 打补丁思路、何时该降 Ubuntu | 编译环境治理 | 25 min |
| 6 | [06-低级失误专题-新手常撞的墙.md](./06-低级失误专题-新手常撞的墙.md) | sudo 编译污染、login 提示符输命令、menuconfig 尺寸、sed 少参数 | 基本功盘点 | 10 min |
| 7 | [07-面试话术-如何把这些经历讲成能力.md](./07-面试话术-如何把这些经历讲成能力.md) | 每个坑对应的面试问题、STAR 话术模板、AI BSP / NPU 岗位映射 | 表达能力 | 20 min |

**建议顺序**:第一次通读按 0→7;查故障时直接跳对应章节;面试前重点看 07。

---

## 三、每章的写作模板

每个坑严格按下面 6 段展开,避免"记流水账":

| 段落 | 内容 | 面试价值 |
|---|---|---|
| **现象** | 原始报错日志 / 现场表现 | 让读者对得上号 |
| **定位思路** | 从日志里哪几行找到线索,先怀疑什么、后怀疑什么 | **分析能力**——面试官最想看的 |
| **根因** | 技术层面到底是什么导致的 | 深度 |
| **解决** | 具体命令 / 补丁 / 配置改动 | 动手能力 |
| **背后原理** | 关联到 SoC 手册 / 内核子系统 / glibc 版本演进 | 知识广度 |
| **岗位映射** | 这个坑锻炼了哪个能力,面试怎么讲 | **岗位贴切** |

---

## 四、覆盖的问题清单

一次性把这份笔记覆盖的所有坑列出来,方便面试前速览:

### Bootloader 层
- U-Boot `bootm c2000000-c4000000` vs `bootm c2000000 - c4000000` (空格分隔的三参数语义)
- `bootcmd` 环境变量丢失 / 变量未定义导致 `bootm` 空参
- TFTP 文件名大小写敏感 + 拼写错误 (`stm32mp157datk.dtb` vs `stm32mp157d-atk.dtb`)
- ARP retry count exceeded (虚拟机 IP 变更 / 网络模式切成 NAT)

### Kernel 启动层
- `VFS: Cannot open root device "nfs" or unknown-block(2,0): error -6` (ENXIO,无网络设备)
- `IP-Config: Complete` 未出现 → 网卡驱动没编进内核
- `stpmic1 0-0033: Unable to read PMIC version` (PMIC I2C 通信 / 电源方案不匹配)
- RTC `Date/Time must be initialized` (掉电、无电池)
- `ALSA device list: No soundcards found` (音频节点未启用,通常可忽略)

### 设备树 & 驱动层
- ATK `stm32mp157d-atk.dtsi` 完全缺失 ethernet0 节点
- SoC 级 `stm32mp157.dtsi` 里外设默认 `status = "disabled"`,必须板级 override
- PHY 型号识别 (YT8511 / RTL8211F / KSZ8081) 与驱动补丁匹配
- Ethernet 驱动 `CONFIG_STMMAC_ETH=y` 与 DTS 使能必须同时满足

### 网络传输层
- NFS v3 vs v4 版本不匹配 → kernel NFS root 走 v3,现代 nfs-kernel-server 默认只开 v4
- `/etc/exports` 缺 `no_root_squash` → 板子作为 root 挂上去访问不了自己文件
- `bootargs ip=` 语法 (client:server:gw:mask:hostname:device:proto)
- U-Boot 里 ping 与 kernel 里网卡通不通是**两码事**(u-boot 有自己的网卡驱动)

### 交叉编译环境层
- `sudo make` 触发 `configure: error: you should not run configure as root` (buildroot 设计禁止 root)
- host-m4 1.4.18 `SIGSTKSZ < 16384` 预处理判断爆炸 (glibc 2.34+ 把 SIGSTKSZ 改成运行期 sysconf 调用)
- host-fakeroot 1.20.2 `_STAT_VER undeclared` (glibc 2.33+ 删除 `__xstat` 内部 API)
- **根本对策**:Ubuntu 24.04 + buildroot 2020.02.6 是不兼容组合,应回退到 Ubuntu 20.04

### 低级失误
- `sudo make` 污染 build 目录所有权
- `menuconfig` 终端窗口小于 19×80 直接报错
- 在 `login:` 提示符敲命令(被当用户名)
- `sed -i '...pattern...' <缺失文件路径>` 报"没有输入文件"

---

## 五、跟我其他仓库的关系

- **[OS_Learn](https://github.com/fyy0619-cell/OS_Learn)**:操作系统五层框架,是理解本仓库的**软件视角基础**。本仓库把那套框架从 RTOS 应用到实际 SoC 上。
- **[SparkLink-FallDetection](https://github.com/fyy0619-cell/SparkLink-FallDetection)**:上层应用项目。本仓库掌握的 BSP/驱动能力,决定了 SparkLink 项目能否顺利落地到自定义板卡。
- **Job_progress (private)**:秋招进度追踪。本仓库的每个坑,都会作为具体 STAR 案例被收录到 Job_progress 的"面试题库"里。

---

## 六、免责与声明

- 所有日志与配置均基于个人开发板复现,非商业环境
- 引用的 ATK 教程属于**正点原子官方资料**,本仓库仅做学习笔记不做二次分发
- 涉及的补丁/命令均以最小侵入方式给出,生产环境请自行评估

---

**最后更新**:2026-09-10
**当前状态**:🚧 持续更新中(方案 C · AI BSP / NPU 复合路,秋招前完成 15+ 篇)
