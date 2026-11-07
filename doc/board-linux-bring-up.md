# 从一块开发板到运行 Linux：Bring-up 流程

本文总结拿到一块 ARM 开发板后，让它稳定运行 Linux 的工程流程。核心原则是：**先建立可观测、可恢复、可复现的最小启动链，再逐层增加功能。**

## 1. 先确认板级事实

在改代码或烧录镜像前，收集并核对：

- SoC 型号、CPU 架构、BootROM 支持的启动介质。
- RAM 型号、容量、DDR 拓扑。
- 启动存储：MicroSD、eMMC、SPI NOR/NAND、NVMe。
- 调试 UART 的引脚、电平、波特率。
- 电源要求、复位键、下载模式、MaskROM/USB OTG。
- 原理图、PCB、SoC 数据手册、厂商 BSP、官方镜像及烧录工具。

对于量产或定制板，原理图和 BSP 至关重要。没有 DDR、PMIC、启动存储连接等信息，无法真正完成底层板级移植。

### 必备工具

- 稳定电源。
- 3.3 V USB-TTL 串口模块。
- 至少两份可替换的启动介质。
- 万用表；必要时使用逻辑分析仪。
- Linux 主机或 Windows + WSL2 交叉编译环境。

### 接线和存储安全

串口只连接：

```text
GND ↔ GND
USB-TTL TX ↔ 板子 RX
USB-TTL RX ↔ 板子 TX
```

- USB-TTL 不向板子供电。
- 不把 5 V 接入 GPIO 或 UART。
- 始终保留一张已验证可启动的基线 SD 卡。
- 早期只从 SD 卡启动；在恢复路径验证前，不写 SPI Flash、eMMC 或 QSPI NOR。

## 2. 先验证官方基线和恢复路径

使用官方或社区维护的已知可用镜像完成以下工作：

1. 从 SD 卡启动并通过串口完整保存上电日志。
2. 登录后记录运行基线：

   ```bash
   uname -a
   cat /proc/cmdline
   cat /proc/device-tree/model
   lsblk
   dmesg -T | tee dmesg-baseline.txt
   ```

3. 记录镜像版本和 SHA-256、主机工具版本、SD 卡型号及烧录方式。
4. 验证能重新烧录 SD 卡，并恢复到该基线镜像。

如果官方镜像都不能启动，优先排查电源、SD 卡、串口接线、启动模式和硬件故障。此时不应开始构建内核或写驱动。

## 3. 拆解真实启动链

典型 ARM Linux 启动路径：

```text
上电 / 复位
  → SoC BootROM
  → SPL / TPL：完成 DDR 初始化
  → U-Boot proper
  → Linux Image + DTB
  → initramfs 或 rootfs
  → /init 或 /sbin/init
  → 用户服务
```

从串口日志中标出每个阶段，并记录组件名称、存储位置、加载地址和启动参数。

### 按日志定位故障层级

| 现象 | 优先排查方向 |
| --- | --- |
| 完全没有输出 | 电源、UART 接线、电平、BootROM、启动介质、SPL、DDR。 |
| 有 U-Boot，内核未启动 | `bootcmd`、内核镜像、DTB、加载地址、`bootargs`。 |
| 卡在 `Waiting for root device` | `root=` 参数、分区 UUID、存储/文件系统驱动、模块是否能加载。 |
| 内核启动但没有登录 | `/init` 或 `/sbin/init`、根文件系统内容、串口 getty。 |

不要同时替换多个组件，否则会失去最小复现。

## 4. 从已有支持开始重建，而不是从零编写

对于已有 Linux 支持的开发板，先从固定版本的 BSP 或上游源码重新构建。按以下顺序替换组件：

```text
厂商 U-Boot + 厂商内核 + 厂商 rootfs
↓
自编 U-Boot + 厂商内核 + 厂商 rootfs
↓
自编 U-Boot + 自编 kernel/DTB + 厂商 rootfs
↓
自编 U-Boot + 自编 kernel/DTB + 自编 rootfs
```

每一步都要能从基线 SD 卡回退。

### 构建纪律

- 固定 U-Boot、内核、Buildroot 的 tag 或 Git commit。
- 记录工具链版本、配置文件和补丁。
- 使用独立输出目录，例如内核的 `O=out/board`，保持源码目录干净。
- 不使用浮动的 `master` 作为可复现基线。
- 每次修改后保存串口日志和 Git commit。

## 5. 先实现最小 Linux 系统

内核阶段只要求以下链路工作：

- 串口 console 输出正常。
- DDR 可用。
- 启动存储可读。
- 根文件系统可挂载。
- 进入 BusyBox shell 或其他最小用户态。

一个最小 rootfs 至少需要：

```text
/init 或 /sbin/init
/bin/sh
/dev/console
/proc
/sys
/tmp
```

当内核执行 `/init` 或 `/sbin/init` 并得到 shell 时，说明 U-Boot、kernel、DTB、内存、console 和 rootfs 主链路已经成立。

## 6. 使用设备树描述板级硬件

设备树不是驱动代码。它描述硬件连接、供电、引脚复用、地址和中断，驱动通过 `compatible` 等属性匹配设备。

一个外设节点示例：

```dts
sensor@76 {
    compatible = "bosch,bme280";
    reg = <0x76>;
    status = "okay";
};
```

实际工作还需要核对：

- 控制器节点是否启用。
- `pinctrl` 是否正确。
- regulator、电源、时钟和 reset GPIO 是否声明。
- 中断线、I2C 地址、SPI mode 和片选是否正确。
- 内核 Kconfig 是否启用了相应控制器和外设驱动。

修改 DTS 后，先看设备和驱动是否出现：

```bash
dmesg -wH
ls /sys/bus/i2c/devices/
ls /sys/bus/spi/devices/
```

## 7. 外设调试：先用户态，后驱动

推荐按以下顺序验证硬件：

1. GPIO：LED 输出、按键输入和中断。
2. I2C：总线、电平、地址、ACK 和寄存器读取。
3. SPI：CS、mode、时钟、MOSI/MISO 和数据帧。
4. UART、RS485、CAN：先确认电气层，再验证协议。
5. 网卡、USB、PCIe、NVMe：最后处理。

### I2C 例子

只在已确认安全的实验总线上执行：

```bash
i2cdetect -l
i2cdetect -y <bus>
i2cget -y <bus> <address> <register>
```

不要扫描 PMIC、EEPROM、未知设备或工业现场设备所在的 I2C 总线。探测命令可能具有副作用。

硬件已由用户态验证后，再写驱动。否则驱动代码只会掩盖接线、电平、地址或时序问题。

## 8. 驱动开发的最小路径

开发一个新外设驱动时：

1. 阅读数据手册：电源、复位、时钟、寄存器、时序、异常状态。
2. 搜索内核是否已有同类驱动及 DT binding。
3. 编写 DTS 节点，确认设备能被创建。
4. 在 `probe()` 中获取 GPIO、regulator、clock、IRQ 等资源。
5. 实现最小数据路径，例如读取 ID 或一个测量值。
6. 处理断线、NACK、超时、资源未就绪和重复 probe。
7. 再添加中断、DMA、缓存、性能优化和电源管理。
8. 接入合适的内核子系统，例如 IIO、input、hwmon、netdev 或 miscdevice。

每个驱动都应明确：设备如何匹配、依赖哪些资源、代码运行在哪种上下文、用户态 ABI 是什么，以及设备异常时如何失败和恢复。

## 9. 用 Buildroot 或 Yocto 固化系统

当启动链和外设已工作，再将系统变为可复现镜像：

- 固定源码版本、工具链、Kconfig 和补丁。
- 用 Buildroot 的 `br2-external`、rootfs overlay、post-build/post-image script 保存定制。
- 不直接修改 Buildroot 的 `output/target`，因为清理构建后改动会丢失。
- 在干净输出目录重建镜像，验证没有依赖手工操作。
- 将 U-Boot、kernel、DTB、rootfs、应用版本写入镜像和启动日志。

## 10. 从“能启动”到“可交付”

一个产品级嵌入式 Linux 系统还需要：

- 应用服务管理、自动重启和健康检查。
- 日志、日志轮转、启动原因和故障码。
- 看门狗、掉电恢复和冷启动测试。
- 外设断线、网络断开、存储写满等故障注入测试。
- 镜像校验、A/B 升级、回滚与受控更新。
- SBOM、CVE 跟踪、许可证和安全启动策略。

最终交付物应包括：

```text
源码版本、配置和补丁
一键构建脚本
烧录与恢复流程
启动日志和存储布局
原理图核对与接线图
外设测试用例
故障矩阵和已知限制
```

## 总结

Bring-up 的正确顺序是：

```text
板级资料与电源
→ 串口与官方基线
→ 启动链定位
→ 自编 U-Boot
→ 自编 kernel + DTB
→ 最小 rootfs
→ 用户态外设验证
→ 驱动开发
→ 可复现构建与产品化
```

每次只改变一个组件。每个阶段保留日志、配置和可回退路径。这样即使启动失败，也能明确问题属于硬件、BootROM/SPL、U-Boot、内核、设备树、根文件系统还是驱动层。
