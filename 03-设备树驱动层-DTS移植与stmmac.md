# 03 · 设备树驱动层 —— DTS 移植与 stmmac

> 本章是**本仓库对 BSP/驱动岗位面试最有分量的一章**。DTS 移植是嵌入式 Linux 移植工程师的核心技能之一,能把 DTS 和驱动 probe 的关系讲清楚,基本能通过技术面。
> 阅读时长:约 25 分钟

---

## 一、先建立基础认知:DTS 三层继承关系

Linux ARM 平台的 DTS 通常按**三层结构**组织:

```
┌────────────────────────────────────────────────┐
│  Level 3  板级 dts                              │  ← 你手上的板子
│  stm32mp157d-atk.dts                            │     加最少的东西,主要指定 model 和 include
├────────────────────────────────────────────────┤
│  Level 2  板级 dtsi (可选,同系列板子共用)      │  ← ATK 板子共用配置
│  stm32mp157d-atk.dtsi                           │     使能哪些外设、PHY 复位脚、电源方案
├────────────────────────────────────────────────┤
│  Level 1  SoC 级 dtsi                           │  ← 芯片厂给的
│  stm32mp157.dtsi → stm32mp153.dtsi              │     所有外设的 reg 地址、中断号、时钟
│    → stm32mp151.dtsi                            │     默认全部 status = "disabled"
└────────────────────────────────────────────────┘
```

**关键设计原则**:SoC 级 dtsi 描述"硬件长啥样",默认全 disable;板级 dtsi 决定"这块板子用哪些",通过 override 打开需要的。

这个设计模式跟 Linux 内核 config 里 `default n` + 板子 `defconfig` 打开的方式**思路完全一致**——芯片能力全描述,板子按需启用。

---

## 坑 3-1:ATK dtsi 里完全没有 ethernet 节点

### 现象

之前 [02 章](./02-Kernel启动层-VFS挂载与rootfs.md) 的坑追到"内核里没有以太网设备",往下查 DTS:

```bash
grep -A 20 "ethernet0\|ethernet@5800a000" arch/arm/boot/dts/stm32mp157d-atk.dts
# 返回空

grep -A 20 "ethernet0\|ethernet@5800a000" arch/arm/boot/dts/stm32mp157d-atk.dtsi
# 也返回空

grep -rn "phy-mode\|phy-handle\|yt8511|YT8511" arch/arm/boot/dts/stm32mp157d-atk.dtsi
# 全部返回空
```

**板级 dtsi 里完全没有 ethernet 相关配置。**

### 定位思路

**先在 SoC 级 dtsi 里确认 ethernet0 的默认状态**:
```bash
grep -B 2 -A 25 "ethernet0:" arch/arm/boot/dts/stm32mp151.dtsi
```
预期看到:
```dts
ethernet0: ethernet@5800a000 {
    compatible = "st,stm32mp1-dwmac", "snps,dwmac-4.20a";
    reg = <0x5800a000 0x2000>, <0x5800c000 0x1000>;
    reg-names = "stmmaceth", "mac-mii";
    interrupts = <0 61 0>;
    ...
    status = "disabled";       ← 默认关闭!
};
```

**结论**:SoC 级已经把节点定义好了(所有寄存器地址、中断都齐全),但状态是 `disabled`。板级 dtsi 没有 override 成 okay,内核就当这块硬件不存在,stmmac 驱动**根本不 probe**。

### 根因

正点原子提供的 dtsi 是"半成品"——只启用了 PMIC、SD/eMMC、UART、USB,**没启用以太网**。可能的原因:
1. ATK 教程的**早期章节 dtsi**,后面章节才会补齐(需要按教程一步步做)
2. ATK 的完整 dtsi 在光盘里,你手上这份是精简版
3. 教程刻意留白让学员练手

### 解决

在 `stm32mp157d-atk.dtsi` 末尾添加(**下面是模板,PHY 相关字段必须按你实际硬件填**):

```dts
&ethernet0 {
    status = "okay";
    pinctrl-0 = <&ethernet0_rgmii_pins_a>;      /* 或对应 rmii 的 pin group */
    pinctrl-1 = <&ethernet0_rgmii_sleep_pins_a>;
    pinctrl-names = "default", "sleep";
    phy-mode = "rgmii-id";                       /* rgmii-id/rgmii/rmii,看 PHY */
    max-speed = <1000>;                          /* 千兆写 1000,百兆写 100 */
    phy-handle = <&phy0>;

    mdio0 {
        #address-cells = <1>;
        #size-cells = <0>;
        compatible = "snps,dwmac-mdio";

        phy0: ethernet-phy@0 {
            reg = <0>;                            /* MDIO 上 PHY 的地址 */
            reset-gpios = <&gpiog 0 GPIO_ACTIVE_LOW>;   /* PHY reset 脚 */
            reset-assert-us = <10000>;
            reset-deassert-us = <300000>;
        };
    };
};
```

**必须搞清楚的硬件信息**:
1. **PHY 芯片型号**(丝印在网卡附近):YT8511C / RTL8211F / KSZ8081 / LAN8720
2. **PHY 接口模式**:RGMII (千兆) / RMII (百兆)
3. **PHY MDIO 地址**:通常 0 或 1,看原理图上 PHYAD 引脚接地/上拉
4. **PHY 复位 GPIO**:哪个 GPIO 输出接 PHY 的 nRST 脚
5. **参考时钟来源**:SoC 出 25MHz 还是 PHY 出

### 特别注意:YT8511 PHY 的驱动问题

**ATK STM32MP157 常用 YT8511C (Motorcomm 裕泰) PHY**,但主线 Linux 5.4 内核**没有 YT8511 驱动**(mainline 到 5.14 才合入 motorcomm.c)。

如果是 YT8511:
1. `.config` 里需要 `CONFIG_MOTORCOMM_PHY=y`(5.14+)
2. 或者手动从 ATK 光盘/上游合入 `drivers/net/phy/motorcomm.c` 补丁到 5.4 内核

**没打这个补丁,即使 stmmac 起来了,PHY probe 会走 Generic PHY fallback,速率可能被限制、link 不稳定。**

### 背后原理:driver probe 是怎么发生的

Linux Kernel 用**总线-设备-驱动**模型:

```
    platform_bus / i2c_bus / spi_bus / pci_bus ...
             ▲                              ▲
     device 注册 (DTS 触发)           driver 注册 (模块加载 / 编入内核)
             │                              │
             └──── compatible 字符串匹配 ───┘
                          │
                     probe() 调用
                          │
                    创建 net_device / char_device / etc.
```

**stmmac 驱动的 probe 触发链**:

1. Kernel 启动时解析 dtb,把 `status = "okay"` 的节点注册为 `platform_device`
2. `stmmac-platform.c` 里 `stmmac_pltfr_driver` 通过 `of_match_table` 声明自己 handle 什么 compatible(如 `"st,stm32mp1-dwmac"`)
3. Kernel 匹配上 → 调用 stmmac 的 `probe` 函数
4. probe 里注册 MDIO bus,扫 PHY,注册 `net_device`
5. `net_device` 出现,`eth0` 名字被分配

**任何一环缺失,链条断掉**:
- DTS status=disabled → 步骤 1 跳过
- 驱动没编 → 步骤 3 跳过
- PHY compatible 不匹配 → 步骤 4 fallback 到 Generic PHY
- MDIO 时钟没配 → 步骤 4 扫不到 PHY

### 岗位映射

**这是 BSP/驱动岗位的核心考点。** 面试话术示例:

> "我遇到过一个板子内核起来但没网卡的问题。系统性排查了三层:
> - kernel .config:确认 CONFIG_STMMAC_ETH=y 编进去了
> - SoC 级 dtsi:ethernet0 节点默认 status=disabled
> - 板级 dtsi:没有 override
>
> 定位到根因是板级 dtsi 缺 ethernet 节点。补上后还有个 PHY 型号问题——ATK 板子用 YT8511,但主线 5.4 没这个 driver,需要从上游 backport motorcomm.c。这个经历让我理解了 Linux driver probe 的完整触发链:总线 device 注册 → driver compatible 匹配 → probe 调用 → 创建具体设备。任何一环断了,ip a 就看不到网卡。"

---

## 坑 3-2:`compatible` 字符串是 SoC 移植的核心

### 一个易错点:compatible 数组匹配规则

DTS 里 compatible 通常写数组:
```dts
compatible = "st,stm32mp1-dwmac", "snps,dwmac-4.20a";
```

驱动 `of_match_table` 里:
```c
static const struct of_device_id stm32_dwmac_match[] = {
    { .compatible = "st,stm32mp1-dwmac"},
    { }
};
```

匹配规则:**从左到右取 compatible 数组的每个字符串,只要有一个能匹配到驱动的 of_match_table 就 probe**。

所以 DTS 里数组前面写"更具体"的、后面写"更通用"的,驱动开发者可以选择匹配层级——细致的匹配走定制驱动,通用的匹配 fallback 到通用驱动。

### 岗位映射

**面试可能问**:"DTS 里 compatible 写两个字符串,内核怎么匹配?"

正确答:"数组从左到右扫,任意一个匹配上驱动的 of_match_table 就 probe。这允许在同一个节点上,既能被专门的板级驱动接管,也能 fallback 到通用驱动。"

**再进阶**:"驱动 probe 时可以通过 `of_device_get_match_data()` 拿到匹配到的具体项,做差异化处理。"

---

## 三、DTS 修改后必须重编 dtb

**新手常犯错误**:改了 dts 但只重编 kernel,忘了重编 dtb。

正确流程:
```bash
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- uImage LOADADDR=0xC2000040 -j$(nproc)
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- dtbs -j$(nproc)
```

然后**两个都要拷贝到 tftpboot**:
```bash
cp arch/arm/boot/uImage /var/lib/tftpboot/
cp arch/arm/boot/dts/stm32mp157d-atk.dtb /var/lib/tftpboot/
```

**验证 dtb 生效的方法**:板子起来后
```bash
cat /proc/device-tree/compatible                    # 看板级 compatible
ls /proc/device-tree/soc/ethernet@5800a000/         # 有说明节点存在
cat /proc/device-tree/soc/ethernet@5800a000/status  # 应该是 "okay\0"
```

---

## 四、DTS 移植的通用方法论(可套用于其他外设)

以 NPU 举例(方案 C 的目标),移植一颗新 NPU 到板子上,DTS 部分要做的事:

| 步骤 | 内容 | 类比本章 |
|---|---|---|
| 1 | 芯片手册确认 NPU 的 register base、中断号、时钟源、复位信号 | 类比 ethernet0 的 reg/interrupts |
| 2 | 供电确认:NPU 是否独立 buck,还是复用主电源 | 类比 STPMIC1 各路 buck 分配 |
| 3 | 检查上游内核有没有 NPU 驱动 | 类比检查 stmmac driver |
| 4 | 写 DTS 节点:compatible、reg、interrupts、clocks、resets、power-domains | 类比 ethernet0 节点 |
| 5 | 编 dtb 并部署,dmesg 看 NPU driver 有没有 probe | 类比 ip a 看有没有 eth0 |
| 6 | 用户态测试:调用 NPU 的 ioctl / mmap 跑最小推理 | 类比 ping 网络 |

**如果面试问"你有没有做过 NPU 移植",你即使没做过,也能说**:
> "STM32MP157 上我没有直接接触 NPU,但我做过 stmmac 网卡的 DTS 移植和 driver probe 调试,方法论是完全通用的:芯片手册确认硬件参数 → DTS 节点描述 → compatible 匹配到 driver → probe 中初始化 → 用户态测试。NPU 无非是把 net_device 换成 char_device,把 PHY 换成 NPU 内部子模块。核心能力(读手册 + 写 DTS + 追 probe 路径)是一样的。"

---

## 本章相关的面试问题

1. **DTS 的三层继承是怎么组织的?板级 dtsi 一般改哪些东西?**
2. **`status = "disabled"` 和 `status = "okay"` 在 kernel 里的实际效果?**
3. **Linux driver probe 的完整触发链?怎么调试 probe 不发生?**
4. **DTS 里 compatible 写多个字符串,匹配规则?**
5. **PHY 芯片没有 mainline 驱动怎么办?**
6. **DTS 改了但不生效,可能的原因?**
7. **如果让你移植一颗新 NPU 芯片到现有板子,DTS 部分你会怎么做?**

**第 7 题基本就是 AI BSP 面试的必问题**,答案套用本章"通用方法论"节。

---

## 下一章

DTS 和驱动 OK 了以后,rootfs 是通过网络传输的,网络这一层还有一堆坑 → [04-网络传输层-TFTP与NFS调试方法论.md](./04-网络传输层-TFTP与NFS调试方法论.md)
