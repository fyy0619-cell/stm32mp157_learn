# 05 · 交叉编译环境 —— Buildroot 与新老系统兼容性

> 本章讲的坑,**看似环境问题,实则考验工程判断力**——什么时候花时间修补丁、什么时候果断换环境。BSP 工程师这个决策每天都在做,面试也常问。
> 阅读时长:约 25 分钟

---

## 一、Buildroot 是什么、能做什么

**一句话**:Buildroot 是从零构建一个嵌入式 Linux 系统的"总包工头"——它自动下载、交叉编译并组装出:
- 交叉编译工具链(如果你不自己提供)
- U-Boot
- Kernel
- rootfs (含 busybox 和你选的所有应用)
- 最终的烧录镜像(sdcard.img / rootfs.tar / uImage / dtb)

**跟其他方案对比**:

| 方案 | 特点 | 学习曲线 | 生产可用 |
|---|---|---|---|
| **Buildroot** | 简单直接,基于 Makefile,一键出镜像 | 中 | 中小型嵌入式产品 |
| **Yocto** | 基于 bitbake,支持复杂 layer 组合 | 高 | 商用 IoT / 汽车 / 工业 |
| **OpenWrt** | 网络设备定向优化 | 中 | 路由器/AP |
| **Debian rootfs + 手动编 uboot/kernel** | 灵活但零散 | 低 | 原型/学习 |

**对 AI BSP 岗位**:Yocto 是大厂常用(Qualcomm、NXP、TI 的 BSP 都基于 Yocto),但学 Buildroot 也不亏——**核心概念(交叉工具链、host tools vs target tools、pkg build 流程)完全通用**。

---

## 坑 5-1:`sudo make` 触发"you should not run configure as root"

### 现象

```bash
sudo make -j8
...
>>> host-tar 1.29 Configuring
...
checking whether mknod can create fifo without root privileges... configure: error: 
you should not run configure as root (set FORCE_UNSAFE_CONFIGURE=1 in environment to bypass this check)
make: *** [package/pkg-generic.mk:259] 错误 1
```

### 根因

**Buildroot 从设计上明确禁止用 root 编译**。理由:
1. 里面很多 GNU 老包(tar、fakeroot 等)的 configure 主动检测并拒绝 root,防止误创建 setuid 文件
2. 编出来的中间产物权限属于 root,后续普通用户 make 会没权限写
3. 潜在的 rm -rf 风险放大

真正需要 root 权限的操作(rootfs 里 mknod 创建设备节点、setuid 位),Buildroot 内部通过 **fakeroot** 用户态模拟解决,全程不需要真 root。

### 解决

**第一步:把之前被 sudo 污染的 build 目录归还给普通用户**:
```bash
sudo chown -R $USER:$USER ~/linux/buildroot/buildroot-2020.02.6/
```

**第二步:清残留**:
```bash
cd ~/linux/buildroot/buildroot-2020.02.6/
rm -rf output/build output/host output/target output/staging output/images
# 保留 dl/ 目录(下载的 tarball,删了要重下)
```

**第三步:不带 sudo 重编**:
```bash
make -j$(nproc)
```

### 岗位映射

**理解"何时需要 root"是嵌入式工程师的基本功。** 面试可能问:"编译时需要 root 权限吗?"标准答案:
- **编译**永远不需要 root(除了极少数 kernel module 装到 /lib/modules/)
- **烧录**可能需要(直接写块设备)
- **运行**在开发板上是 root(单用户系统);在桌面 Linux 上不需要

区分这三个场景就能避免 90% 的权限相关误操作。

---

## 坑 5-2:host-m4 `SIGSTKSZ < 16384` 编译错误

### 现象

```
c-stack.c:55:26: error: missing binary operator before token "("
   55 | #elif HAVE_LIBSIGSEGV && SIGSTKSZ < 16384
      |                          ^~~~~~~~
```

### 定位思路

**关键判断:这是"代码写错了"还是"环境不兼容"?**

看代码:`#elif HAVE_LIBSIGSEGV && SIGSTKSZ < 16384` —— 这在 M4 1.4.18 (2016 年) 里跑了 8 年都没问题,突然报错。**十有八九是环境变化,不是代码问题**。

搜索 SIGSTKSZ 在什么时候变化过 → **glibc 2.34 (2021)** 把 `SIGSTKSZ` 从编译期常量改成运行期 `sysconf(_SC_SIGSTKSZ)` 调用,预处理器再也不能拿它跟数字比较。

### 根因

Ubuntu 24.04 用 glibc 2.39,SIGSTKSZ 已经不是编译期常量。Buildroot 2020.02.6 里带的 M4 1.4.18 假设它是常量,预处理 `#elif ... SIGSTKSZ < 16384` 展开后变成 `#elif ... sysconf(_SC_SIGSTKSZ) < 16384`——这是运行期代码,预处理器爆炸。

### 解决

一行 sed 打补丁:
```bash
sed -i 's/#elif HAVE_LIBSIGSEGV && SIGSTKSZ < 16384/#elif HAVE_LIBSIGSEGV/' \
    output/build/host-m4-1.4.18/lib/c-stack.c
make -j$(nproc)
```

意思是删掉那段有毒的条件,只保留 `#elif HAVE_LIBSIGSEGV`。M4 本身不太依赖这个精细分支,能过。

### 背后原理:glibc ABI 演进

glibc 每几年会做**破坏性 API 变更**,常见的:

| glibc 版本 | 破坏点 | 影响 |
|---|---|---|
| 2.28 (2018) | `libio.h` 移除 | gnulib / m4 大量 breakage |
| 2.32 (2020) | `sys_siglist` deprecated | 大量老程序警告 |
| 2.33 (2021) | 移除 `_STAT_VER` 和 `__xstat` 系列 | fakeroot / dpkg / 老包全部失败 (见坑 5-3) |
| 2.34 (2021) | `SIGSTKSZ` 变成 sysconf | 本坑 |
| 2.34 (2021) | `libpthread` / `libdl` 合入 `libc` | 老 Makefile 链接失败 |
| 2.36 (2022) | `arpa/nameser_compat.h` 变化 | net-tools 类问题 |

**这些是发行版升级带来的隐性成本**。BSP 工程师必须知道自己 target 用的 glibc 版本、编译宿主的 glibc 版本,以及两者不一致时的雷区。

### 岗位映射

**这个坑体现的能力:**
- 阅读预处理器错误的能力
- 追溯"什么时候变了"的能力
- 打最小侵入补丁的能力

面试话术:
> "编 buildroot 时撞到 M4 的一个 SIGSTKSZ 预处理错误。我没直接搜错误信息求答案,而是先想'这段代码好几年了为什么突然错',反向查 SIGSTKSZ 演进,定位到 glibc 2.34 把它从编译期常量改成 sysconf。打个最小补丁——只删条件里破损的部分,保留主逻辑——过了。这种问题背后是 glibc ABI 每几年一次的破坏性变更,BSP 工程师要熟悉这个演进史,选宿主 Ubuntu 版本时才不会踩坑。"

---

## 坑 5-3:host-fakeroot `_STAT_VER undeclared`

### 现象

```
libfakeroot.c: In function 'chown':
libfakeroot.c:99:40: error: '_STAT_VER' undeclared (first use in this function)
   99 | #define INT_NEXT_STAT(a,b) NEXT_STAT64(_STAT_VER,a,b)
      |                                        ^~~~~~~~~
...
（十几个函数全炸:chown/lchown/fchown/chmod/mkdir/unlink/rename/... setxattr/getxattr/...）
```

### 根因

**glibc 2.33 (2021) 移除了 `_STAT_VER` 符号和整套 `__xstat` 内部 API**。

历史背景:glibc 早期的 `stat/lstat/fstat` 是通过 `__xstat/__lxstat/__fxstat` 加一个版本号 `_STAT_VER` 实现的(为了兼容 struct stat 的 layout 变化)。2.33 认为这个抽象层没意义了,直接删掉——用户代码用 `stat()` 会走新的系统调用直接实现。

**fakeroot 1.20.2 恰好 hook 的是 `__xstat` 这套内部 API**,而不是 `stat()`。glibc 一改,fakeroot 直接崩。

### 修复代价

跟坑 5-2 (一行 sed) 不一样,这个需要:
- 把十几个 hook 函数从 `__xstat64` 改成新的 `stat64` 调用
- 处理 `struct stat` 变化
- 或者从上游 backport fakeroot 后续版本的补丁

**这属于"改动量较大"级别的补丁**,可能改几百行代码。

### 解决(如果坚持不换环境)

上游 fakeroot 1.24+ 已经修了这个问题。可以尝试:
```bash
# 让 buildroot 用宿主系统的 fakeroot,不编 host-fakeroot
sudo apt install fakeroot
```
然后修改 buildroot 让它跳过 host-fakeroot 编译(需要改 mk 文件)——**这条路我没走通,不推荐**。

**推荐做法:直接换 Ubuntu 20.04**(见下节)。

---

## 二、工程判断:何时打补丁 vs 何时换环境

BSP 工程师最重要的能力之一是**这类决策**。判断标准:

### 打补丁的情境
- **单点问题**,补丁一两行代码
- 触碰的是**边缘功能**(比如警告、非关键路径)
- 后续能保持环境不变继续工作

### 换环境的情境
- **系统性不兼容**,一个坑接一个坑
- 补丁需要**改核心代码**(hook 函数、系统调用抽象)
- 已经投入 X 小时还看不到底

**本项目的判断**:Ubuntu 24.04 + Buildroot 2020.02 是系统性不兼容(前两个坑已经暴露:m4 + fakeroot),预期后面还有 python2 / perl / kernel-headers 一堆。**果断换 Ubuntu 20.04,而不是硬扛**。

### 岗位映射(极重要)

**面试话术示范**:

问:"如果编译环境和源码不匹配,你怎么处理?"

答:
> "分两种情况判断。如果是单点问题,补丁一两行,我会打补丁保持环境。但如果是系统性不兼容——比如 Ubuntu 24 + Buildroot 2020,glibc 演进导致 m4、fakeroot、gettext、bison 一路都是坑——我会果断换环境。理由是我的时间应该花在业务问题上,不是 4 年前 Buildroot 的兼容维护。BSP 工程师的价值在于'从上电到跑起来',不在于'给老 buildroot 缝缝补补'。当然生产环境如果被绑死在某个宿主版本,那另说,那时才该硬扛。"

**这个回答的加分点**:
- 能给出**判断标准**,不是"看情况"
- 能算**时间账**,体现工程 sense
- 能区分**开发环境 vs 生产环境**,体现成熟度

---

## 三、宿主 Ubuntu 版本选型指南

| Ubuntu 版本 | glibc | 支持的 Buildroot | 备注 |
|---|---|---|---|
| 18.04 | 2.27 | 2018.x - 2021.x | 已 EOL,不推荐 |
| **20.04** | **2.31** | **2019.x - 2022.x** | **老 BSP 项目最佳** |
| 22.04 | 2.35 | 2022.x - 2024.x | 中间地带 |
| **24.04** | **2.39** | **2024.x+** | **必须配新 Buildroot** |

**通用原则**:宿主 Ubuntu 和 Buildroot 发行**同年或差一年内**,基本没兼容问题。差三年以上,做好打补丁准备。

**对 ATK STM32MP157 项目**:教程基于 Ubuntu 20.04 + Buildroot 2020.02,严格贴合就走 20.04,不要挑战。

---

## 四、Buildroot 关键目录速览

熟悉这几个目录能定位 80% 的问题:

```
buildroot-2020.02.6/
├── configs/                    ← 板级 defconfig,如 stm32mp157d_atk_defconfig
├── package/                    ← 每个包一个子目录,含 Config.in 和 .mk
│   ├── m4/
│   │   ├── Config.in           ← 编不编选项
│   │   ├── m4.mk               ← 版本、下载 URL、编译规则
│   │   └── 0001-xxx.patch      ← 打补丁
│   └── busybox/
├── output/                     ← 编译中间产物
│   ├── build/                  ← 每个包解压+编译在这
│   │   └── host-m4-1.4.18/     ← 撞坑时看这里的源码
│   ├── host/                   ← host 工具(交叉编译器等)
│   ├── target/                 ← target rootfs 的原样
│   ├── staging/                ← target 头文件+库(链接用)
│   └── images/                 ← 最终镜像 (sdcard.img, rootfs.tar 等)
├── dl/                         ← 下载的 tarball 缓存
└── Makefile                    ← 顶层 Makefile
```

**打补丁的正确姿势**(如果决定长期维护):
1. 在 `package/m4/` 下创建 `0003-fix-sigstksz.patch`
2. Buildroot 会自动检测并应用
3. 下次 `make clean && make` 补丁会自动打上

**临时改法**(赶时间):
- 直接改 `output/build/host-m4-1.4.18/` 里的源码
- 但 `make clean` 后会消失

---

## 五、跟工具链的关系

Buildroot 里的**交叉工具链**可以来自三种源:

1. **Buildroot 自己编 (Buildroot Toolchain)**:最灵活,时间最久
2. **外部预编译 (External Toolchain)**:如 ARM 官方 gcc-arm 系列,快
3. **系统的 gcc-arm-linux-gnueabihf apt 包**:够用但版本旧

**ATK 教程用的是外部工具链**:`gcc-arm-9.2-2019.12-x86_64-arm-none-linux-gnueabihf`——需要单独下载,解压后加 PATH。

**面试可能问**:"交叉工具链跟宿主 gcc 有什么区别?" 答:
- 宿主 gcc 编 x86_64 代码,链接 x86_64 glibc
- 交叉 gcc 编 arm 代码,链接 arm glibc / musl
- 命名规范 `<arch>-<vendor>-<sys>-<abi>-gcc`,如 `arm-none-linux-gnueabihf`
- **sysroot** 概念:交叉编译时头文件和库必须来自 target,不能用宿主的

---

## 本章相关的面试问题

1. **Buildroot 和 Yocto 你会怎么选?**
2. **为什么编 Buildroot 不能用 sudo?fakeroot 怎么工作的?**
3. **glibc 2.34 之后 SIGSTKSZ 变化你了解吗?为什么 buildroot 老包会崩?**
4. **宿主 Ubuntu 版本和 buildroot 版本不匹配,你怎么权衡打补丁 vs 换环境?**
5. **交叉工具链的 sysroot 是什么?为什么必须用 target 的头文件?**
6. **Buildroot output 目录下 target / staging / host 三个子目录的区别?**

---

## 下一章

工程问题聊完,下面是一些**低级失误**的复盘,这类坑不难但能暴露基本功 → [06-低级失误专题-新手常撞的墙.md](./06-低级失误专题-新手常撞的墙.md)
