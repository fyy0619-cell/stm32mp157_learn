# 04 · 网络传输层 —— TFTP 与 NFS 调试方法论

> 板子和虚拟机通过网络传 kernel/dtb/rootfs,这条链路细节多、坑也多。本章重点讲**分层排查的方法论**,而不是罗列具体命令。
> 阅读时长:约 20 分钟

---

## 一、开发链路全景

```
        Windows 主机
        192.168.5.100
              │
        物理网卡 (桥接)
              │
        ┌─────┴─────┐
        │           │
   Ubuntu VM   STM32MP157 板子
   192.168.5.10    192.168.5.2
   (TFTP + NFS)    (U-Boot / Kernel)
        │           │
        └───同一网段─┘
```

**核心约定**:三者必须在同一 IP 网段(这里都是 192.168.5.x),网关一致,虚拟机走桥接不能 NAT。

---

## 二、通用分层排查表(必须背下来)

调网络问题**永远从 OSI 底层往上排**,一层一层验证:

| OSI 层 | 检查内容 | 检查命令 | 通过标志 |
|---|---|---|---|
| L1 物理 | 网线接好、网卡灯亮 | 目视 | 灯常亮/闪烁 |
| L2 数据链路 | ARP 表能建立 | U-Boot 里 `ping <server>` | 有响应 |
| L3 网络 | IP 层能通 | `ping 192.168.5.10` | 有回复 |
| L4 传输 | 端口能连接 | Ubuntu 里 `nc -uz 192.168.5.10 69`(TFTP) | 无 refused |
| L7 应用 | 业务协议正确 | `tftp` 拉具体文件 / `showmount -e` | 数据可传 |

**遇到 network 类问题,80% 都能靠这个表快速定位。**

---

## 坑 4-1:NFS v3 vs v4 版本不匹配(最隐蔽的坑)

### 现象

bootargs 明确写了 NFS,IP 也配对了,但 Kernel 挂 rootfs 失败,可能报:
```
VFS: Unable to mount root fs via NFS
NFS: server 192.168.5.10 not responding
```
或者更隐蔽的:client 端 syslog 里没错,server 端 nfsd 也没错,但就是挂不上。

### 定位思路

**先在 Ubuntu 上确认 NFS 支持哪些版本**:
```bash
cat /proc/fs/nfsd/versions
```

新版 Ubuntu (20.04+) 默认输出可能是:
```
-2 +3 +4 +4.1 +4.2
```
`-` 号表示**禁用**。有些发行版默认只启 v4,v3 是关的。

### 根因

Kernel 里 NFS root(`root=/dev/nfs`)默认走 **NFSv3**(部分老内核甚至 v2)。如果 server 端只开 v4,自然挂不上。

### 解决

**Server 端启用 v3**——编辑 `/etc/nfs.conf`:
```
[nfsd]
vers3=y
vers4=y
```
重启:
```bash
sudo systemctl restart nfs-kernel-server
```

**或者在 bootargs 里显式指定 v3**:
```
nfsroot=192.168.5.10:/home/fu/nfs/rootfs,nfsvers=3,proto=tcp
```

### 岗位映射

**这个坑体现的能力:知道协议有多版本、能验证 server/client 协商结果。**

面试话术:"NFS root 走 v3 但现代 server 默认只开 v4,这种协议版本不匹配问题在 SMB、HTTP/2、gRPC 里都会遇到。我的排查习惯是先分别 dump 双方支持的版本,而不是猜网络不通。"

---

## 坑 4-2:`/etc/exports` 少 `no_root_squash`

### 现象

Server 端一切看起来正常,client 板子挂上后:
- 能 ls 目录
- 但读写自己 rootfs 文件报权限错
- 或者 init 起来后立刻 Kernel panic(因为 root 身份读不了 /sbin/init)

### 定位思路

嵌入式 Linux 的 root 文件系统里,几乎所有文件的 owner 都是 root。板子作为 root 挂 NFS 后,如果 server 端把 root 映射成 nobody(root_squash 是默认行为),等于**板子看到的 rootfs 大部分文件都是"nobody 拥有的 root 文件"**,访问一堆权限拒绝。

### 根因

NFS server 默认开启 `root_squash`——把 client 的 root 用户映射成 anonymous (nobody),防止 client 提权访问 server 敏感文件。**对开发用途完全不适用**。

### 解决

`/etc/exports` 必须加 `no_root_squash`:
```
/home/fu/nfs/rootfs *(rw,sync,no_root_squash,no_subtree_check)
```
应用:
```bash
sudo exportfs -arv
```

**各选项含义速记**:
- `rw`:读写(必须)
- `sync`:同步写(数据安全,性能一般)
- `no_root_squash`:不做 root 降权(开发必须)
- `no_subtree_check`:关闭子树检查(避免奇怪的 bug)

### 岗位映射

**理解安全默认与开发便利的 trade-off。**

面试问"为什么 NFS 默认 root_squash?什么时候要关?" 答:
- 默认开是安全考虑,避免 client root 提权
- 开发/嵌入式场景必须关,因为 client 就是 root 在用,还要跑 init/驱动加载,squash 后没法工作
- 生产环境如果一定要 root_squash,一般用 `anonuid=<uid>` 强制映射到指定用户

---

## 坑 4-3:`bootargs ip=` 语法一个字符错就废

### 现象

bootargs 里的 `ip=` 写错,kernel 起来后 dmesg 里没有 `IP-Config: Complete` 这行,NFS 根本连不上。

### `ip=` 语法完整格式

```
ip=<client-ip>:<server-ip>:<gateway>:<netmask>:<hostname>:<device>:<autoconf>
```

**七个字段用冒号分隔,一个都不能少**(可以为空但冒号必须在)。字段含义:

| 字段 | 说明 | 常用值 |
|---|---|---|
| client-ip | 板子 IP | 192.168.5.2 |
| server-ip | NFS server IP | 192.168.5.10 |
| gateway | 网关 | 192.168.5.1 |
| netmask | 掩码 | 255.255.255.0 |
| hostname | 主机名 | (通常空) |
| device | 网卡名 | eth0 |
| autoconf | 自动配置协议 | off / on / dhcp |

**完整示例**:
```
ip=192.168.5.2:192.168.5.10:192.168.5.1:255.255.255.0::eth0:off
```
注意 hostname 空但 `::` 必须保留。`autoconf=off` 表示"用上面填的静态 IP,不要 DHCP"。

### 常见错误

1. **少一个冒号** → 后面所有字段全错位
2. **`autoconf=on` 但没 DHCP server** → hang 住等 DHCP
3. **写错 device 名(eth0 vs enp0s0 等)** → 找不到网卡

### 岗位映射

**理解 kernel command line 的字段化配置模式。**

同样的字段化字符串在 kernel `console=`、`rootflags=`、`modprobe.blacklist=` 里都有,搞清楚一处的语法逻辑,其他都能类推。

---

## 坑 4-4:U-Boot 能 ping 通 ≠ Kernel 能 ping 通

### 现象

U-Boot 里:
```
STM32MP> ping 192.168.5.10
ping 192.168.5.10 is alive
```
Kernel 起来后 (假设能挂上 rootfs):
```
# ping 192.168.5.10
ping: sendmsg: Network unreachable
```

### 根因

**U-Boot 和 Kernel 是两个完全独立的软件栈,各自有自己的**:
- 网卡驱动
- IP 协议栈
- ARP 表
- 路由表

U-Boot 能 ping 通只证明:
- 物理链路 OK
- PHY 供电 OK
- 交换机/网络设备转发 OK

**说明不了 Kernel 里的驱动 probe 是否成功、网卡配置是否正确、路由是否加了。**

### 排查

在 Kernel 里:
```bash
ip link                        # 网卡设备存在?
ip addr                        # 有 IP?
ip route                       # 有默认路由?
ethtool eth0                   # 链路 up?
dmesg | grep -i "eth\|phy"     # 有报错?
```

### 岗位映射

**这是最典型的"两个世界隔离"问题**。BSP 工程师必须清楚:U-Boot 里能做的事,Kernel 里可能重头再来一遍。

同类现象:
- U-Boot 里能读 MMC ≠ Kernel 里 SD 卡驱动能用
- U-Boot 里 GPIO 输出 OK ≠ Kernel 里 GPIO 驱动能用
- U-Boot 里 I2C 通信 OK ≠ Kernel i2c-tools 能用

**每一层的驱动栈独立,是嵌入式 Linux 相比 MCU 单一固件模型最大的复杂度来源。**

---

## 二、NFS 挂载排查决策树(收藏用)

```
NFS 挂不上
   │
   ├─ Kernel dmesg 有 IP-Config: Complete 吗?
   │   ├─ 没有 → 网卡驱动/DTS 问题 (回 [03 章])
   │   └─ 有 → 继续
   │
   ├─ 板子 ping 通 server 吗?
   │   ├─ 不通 → 物理层/路由问题
   │   └─ 通 → 继续
   │
   ├─ server 端 showmount -e 能列出 rootfs 目录吗?
   │   ├─ 不能 → /etc/exports 没生效,exportfs -arv
   │   └─ 能 → 继续
   │
   ├─ /etc/exports 里有 no_root_squash 吗?
   │   ├─ 没有 → 加上重启 nfs-server
   │   └─ 有 → 继续
   │
   ├─ server 支持 NFS v3 吗?
   │   ├─ 不支持 → nfs.conf 开 vers3=y 或 bootargs 强制 v4
   │   └─ 支持 → 继续
   │
   ├─ bootargs 里 ip= 七个字段完整吗?
   │   ├─ 不完整 → 补齐
   │   └─ 完整 → 继续
   │
   └─ 防火墙关了吗?
       ├─ 没关 → sudo ufw disable
       └─ 关了 → 抓包 (tcpdump) 看具体协议交互
```

**这棵树背下来,面试问"NFS 挂不上你怎么查",能一口气讲完 6 层,足够加分。**

---

## 本章相关的面试问题

1. **网络问题你的排查方法论是什么?**
2. **NFS v3 和 v4 主要区别?为什么内核 NFS root 走 v3?**
3. **`no_root_squash` 是什么?为什么开发环境必须开?**
4. **kernel command line 的 `ip=` 语法你熟吗?哪几个字段?**
5. **U-Boot 里能 ping 通,Kernel 里就一定能 ping 通吗?为什么?**
6. **TFTP 和 NFS 都是 UDP-based,但 NFS 也支持 TCP,什么场景该用哪个?**

---

## 下一章

网络问题解决,rootfs 挂上了,但 rootfs 本身要 buildroot 编出来,编译环境有一大堆坑 → [05-交叉编译环境-Buildroot新老系统兼容性.md](./05-交叉编译环境-Buildroot新老系统兼容性.md)
