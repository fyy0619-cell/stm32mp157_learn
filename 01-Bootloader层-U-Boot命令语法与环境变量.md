# 01 · Bootloader 层 —— U-Boot 命令语法与环境变量

> 本章覆盖 4 个 U-Boot 阶段的坑,全部是**"命令语义没搞清楚"或"环境变量没同步"**导致的。共同点:硬件、内核、rootfs 都是完好的,坑纯粹出在 Bootloader 交互上。
> 阅读时长:约 20 分钟

---

## 坑 1-1:`bootm c2000000-c4000000` —— 少了两个空格

### 现象

```
STM32MP> boot
...
7313912 bytes read in 193 ms       ← kernel 加载成功
63881 bytes read in 30 ms          ← dtb 加载成功
bootm                              ← 然后突然打帮助
Usage:
bootm [addr [arg ...]]
    - boot application image stored in memory
        passing arguments 'arg ...'; ...
STM32MP>                           ← 掉回命令行
```

文件都读进内存了,`bootm` 却打帮助信息,直接掉回 U-Boot shell。

### 定位思路

**第一反应不是"bootm 坏了",而是"bootm 收到了什么参数"。** 查 `bootcmd`:

```
STM32MP> printenv bootcmd
bootcmd=ext4load mmc 1:2 c2000000 uImage;ext4load mmc 1:2 c4000000 stm32mp157d-atk.dtb;bootm c2000000-c4000000
```

问题一眼看到:`c2000000-c4000000` **没有空格**。

### 根因

U-Boot 的 `bootm` 语法是:
```
bootm <kernel_addr> <initrd_addr> <fdt_addr>
```
**三个参数用空格分隔**,不用 initrd 就用 `-` 占位。所以正确写法是:
```
bootm c2000000 - c4000000
```

写成 `c2000000-c4000000` 会被 U-Boot 当成**一个**参数——一串看起来像 hex 但实际不合法的字符串。U-Boot 只收到 1 个参数,认为语法错误,打帮助。

### 解决

```
setenv bootcmd 'ext4load mmc 1:2 c2000000 uImage;ext4load mmc 1:2 c4000000 stm32mp157d-atk.dtb;bootm c2000000 - c4000000'
saveenv
reset
```

### 背后原理

U-Boot 是宏语言解释器,**参数分隔靠空格**(而不是像 C 语言那样靠逗号或者括号)。这跟大部分 shell 保持一致。

`-` 单字符在 U-Boot 命令里通常代表"跳过这个位置的参数",是**保留占位符**。同类语义在 `bootz`、`booti`、`bootl` 命令里都一样。

### 岗位映射

**这个坑锻炼的能力:阅读 U-Boot 官方文档 + 从帮助信息反推参数格式。**

面试话术示例:
> "有次调启动脚本,发现内核和 dtb 都加载成功了,但 bootm 直接打帮助信息掉回 shell。我第一反应是不去猜 bootm 坏没坏,而是看它到底收到了什么参数。printenv 看 bootcmd 立刻发现 `c2000000-c4000000` 少了空格——被 U-Boot 当成一个 token,所以 bootm 只收到一个参数。这个坑让我养成了一个习惯:遇到 bootloader 命令异常,先看环境变量和命令历史,而不是直接怀疑命令本身。"

---

## 坑 1-2:`bootcmd` 里的变量没定义

### 现象

跟坑 1-1 长得几乎一样:

```
7313912 bytes read in 193 ms
63881 bytes read in 30 ms
bootm
Usage: bootm [addr [arg ...]]
```

### 定位思路

同样 `printenv bootcmd`,但这次结果是:

```
bootcmd=ext4load mmc 1:2 ${kernel_addr_r} uImage;ext4load mmc 1:2 ${fdt_addr_r} dtb;bootm ${kernel_addr_r} - ${fdt_addr_r}
```

用了变量 `${kernel_addr_r}` 和 `${fdt_addr_r}`,再查它们:

```
STM32MP> printenv kernel_addr_r fdt_addr_r
## Error: "kernel_addr_r" not defined
## Error: "fdt_addr_r" not defined
```

**变量是空的**,所以 `bootm ${kernel_addr_r} - ${fdt_addr_r}` 展开后变成 `bootm  -  `,U-Boot 收到 0 个有效地址,又打帮助。

### 根因

U-Boot 环境变量在几种情况下会消失:
1. **首次烧录 U-Boot 后没 `saveenv`**:内存里的默认值没写到 MMC 环境分区
2. **环境分区被覆盖**:烧其他镜像时误擦
3. **U-Boot 编译时的默认 env 里没定义**:有些精简版 U-Boot 不带 `kernel_addr_r`

### 解决

**先临时定义验证**:
```
setenv kernel_addr_r 0xc2000000
setenv fdt_addr_r 0xc4000000
setenv ramdisk_addr_r 0xc4400000
saveenv
reset
```

**永久修复**:也可以直接写死地址,不用变量:
```
setenv bootcmd 'ext4load mmc 1:2 c2000000 uImage;ext4load mmc 1:2 c4000000 stm32mp157d-atk.dtb;bootm c2000000 - c4000000'
saveenv
```

### 背后原理

U-Boot 环境变量分**两个存储层次**:

1. **默认环境** (编译时嵌入 U-Boot 二进制的 `env_default` 数组)
2. **持久化环境** (从存储介质加载,通常在 MMC 的独立分区)

启动时:先加载默认,再用持久化覆盖。`saveenv` 只写持久化那份;`env default -a` 恢复到默认。

### 岗位映射

**锻炼的能力:理解嵌入式系统"多层配置覆盖"的通用模式。**

这个模式在 Kernel(默认 config + 命令行覆盖)、DTS(dtsi + dts override)、systemd (default + drop-in)、Android property (build.prop + persist.prop) 里都一样。

---

## 坑 1-3:TFTP 文件名拼错

### 现象

```
STM32MP> boot
ethernet@5800a000 Waiting for PHY auto negotiation to complete... done
Using ethernet@5800a000 device
TFTP from server 192.168.5.10; our IP address is 192.168.5.2
Filename 'uImage'.
Load address: 0xc2000000
Loading: #################################################...
         698.2 KiB/s
done
Bytes transferred = 7312840 (6f95c8 hex)      ← kernel 拉成功
Using ethernet@5800a000 device
TFTP from server 192.168.5.10; our IP address is 192.168.5.2
Filename 'stm32mp157datk.dtb'.                 ← 但 dtb 找不到
Loading: *
TFTP error: 'File not found' (1)
Not retrying...
## Booting kernel from Legacy Image at c2000000 ...
   ...
ERROR: Did not find a cmdline Flattened Device Tree
   XIP Kernel Image                            ← 内核起不来
```

### 定位思路

**看 TFTP error 的具体信息:`'File not found' (1)`。** 这明确告诉你:网络通了、TFTP 服务响应了、只是**服务端目录里没这个文件**。

对比:
- 之前刚烧到 MMC 时 ext4ls 看到的文件名:`stm32mp157d-atk.dtb`(**有横线**)
- bootcmd 里写的:`stm32mp157datk.dtb`(**没横线**)

**手滑掉了一个字符**——setenv 时候记错了。

### 根因

TFTP 是**大小写敏感 + 字符精确匹配**协议。多一个/少一个字符、大小写不同,都会 File not found。

### 解决

改 bootcmd,并注意加 `&&` 让加载失败时不启动(避免 XIP 误判):
```
setenv bootcmd 'tftp c2000000 uImage && tftp c4000000 stm32mp157d-atk.dtb && bootm c2000000 - c4000000'
saveenv
```

服务端也检查一下:
```bash
ls -la /var/lib/tftpboot/
```

### 背后原理

**为什么后续 bootm 还是执行了?** 因为你原来的 bootcmd 用的是 `;` 分隔,U-Boot 里 `;` 表示"顺序执行,不看返回值"——第一条 tftp 失败,第二条 tftp 也失败,第三条 bootm 照样跑,拿着 `0xc4000000` 处的**上一次残留数据**当 fdt 去解析,当然找不到 device tree magic (`0xd00dfeed`)。

改成 `&&` 后语义变成"上一条成功才执行下一条",能提前中断。

### 岗位映射

**锻炼的能力:理解 shell/脚本语义细节,以及"部分成功"是最难调的 bug。**

面试话术示例:
> "TFTP 拉 dtb 失败后,U-Boot 用 c4000000 处的旧数据当 fdt 硬跳内核,报了个 XIP 的错。这种'半死不活'的 bug 特别难调,因为看起来内核开始启动了。我把 bootcmd 里的 `;` 换成 `&&`,让任何一步失败都立刻中断,后面类似的 bug 全都会在正确的位置报错。工程里防止'部分成功'比修具体 bug 更重要。"

---

## 坑 1-4:`ARP Retry count exceeded`

### 现象

之前 TFTP 能通,这次开机变成:

```
TFTP from server 192.168.5.10; our IP address is 192.168.5.2
Filename 'uImage'.
Load address: 0xc2000000
Loading: *
ARP Retry count exceeded; starting again
Using ethernet@5800a000 device
TFTP from server 192.168.5.10; our IP address is 192.168.5.2
Filename 'stm32mp157d-atk.dtb'.
Loading: *
ARP Retry count exceeded; starting again
Wrong Image Format for bootm command
ERROR: can't get kernel image!
```

**`Loading: *` 后立刻 ARP 超时**——只发出了一个字符就再也没进展。

### 定位思路

关键在 `ARP Retry count exceeded`。这告诉你**问题在二层(数据链路层)**——U-Boot 发 ARP 请求问"谁的 IP 是 192.168.5.10",没人应答。

**注意:这不是 TFTP 服务的问题,是 IP-to-MAC 都还没解析成功。**

分层排查:
1. 物理层:网线松了 / 网卡灯不亮?
2. 网络配置:虚拟机 IP 是不是真的 192.168.5.10?
3. 虚拟机网络模式:桥接还是 NAT?

### 根因(实际命中的)

**虚拟机网络设置被切成 NAT 或者 IP 变了。** 常见场景:
- VMware 挂起后恢复,网络配置漂了
- Ubuntu 上 systemd-networkd 或 NetworkManager 抢了网卡控制权,重分配 IP
- Windows 主机换了网卡(比如有线切无线),桥接的物理网卡变了

### 解决

先在 U-Boot 里 ping 验证:
```
ping 192.168.5.10
```
- ping 通 → 问题在 TFTP 服务本身,查 `systemctl status tftpd-hpa`
- ping 不通 → 二层不通,按下面步骤:

在虚拟机里查真实 IP:
```bash
ip addr | grep -A2 "ens\|eth\|enp"
```

如果 IP 不对,两种改法:
```bash
# 改虚拟机 IP 静态化
sudo vim /etc/netplan/01-network-manager-all.yaml
sudo netplan apply
```
或者改 U-Boot 的 serverip:
```
setenv serverip <虚拟机新IP>
saveenv
```

**VMware 网络模式**必须是 **Bridged**(桥接),不能 NAT——NAT 模式下虚拟机在自己的私网,板子通过物理网口过来根本发现不了它。

### 背后原理

**ARP 是 IP → MAC 的地址解析,工作在 OSI L2**。TFTP/FTP/HTTP 都是 L7,任何 L7 通信之前必须 L2/L3 先通。

嵌入式调网,**永远按 OSI 从下往上排**:
1. 物理层:线、灯
2. 数据链路层:ARP 能不能通(ARP 表可以查)
3. 网络层:ping
4. 传输层:telnet 端口
5. 应用层:业务命令

### 岗位映射

**锻炼的能力:OSI 分层排查思维、虚拟机网络配置基础。**

这是**面试网络类问题的经典切入点**。对方问"你怎么排查网络故障?"直接答"从 OSI 底层往上排",立刻把你和"只会 ping 一下就说不通"的候选人区分开。

---

## 本章相关的面试问题

准备面试时可以自问自答:

1. **U-Boot 里 `bootm` 的三个参数分别是什么?为什么中间要用 `-` 占位?**
2. **`bootcmd` 里 `;` 和 `&&` 的区别?什么场景该用哪个?**
3. **U-Boot 环境变量存在哪里?什么情况下会丢?**
4. **TFTP 拉不到文件,可能有哪几层原因?怎么快速定位?**
5. **`ARP Retry count exceeded` 说明卡在 OSI 哪一层?接下来怎么查?**
6. **U-Boot 里能 ping 通服务器,是不是说明 Kernel 里网卡也能用?为什么?**

**第 6 题特别重要**——很多人会答"能",但正确答案是"不能,因为 U-Boot 和 Kernel 用完全独立的驱动栈,U-Boot 里网卡工作说明硬件和 PHY 供电 OK,但 Kernel 里的 stmmac 驱动 probe 是完全独立的过程。"

---

## 下一章

Bootloader 完事之后进入内核,如果 rootfs 挂不上就是 Stage 4 末尾的坑 → [02-Kernel启动层-VFS挂载与rootfs.md](./02-Kernel启动层-VFS挂载与rootfs.md)
