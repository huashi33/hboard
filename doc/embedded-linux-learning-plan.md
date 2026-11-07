# Orange Pi 嵌入式 Linux 学习计划

> 适用对象：电子信息工程专业，7 年 C/C++ 与工业 SCADA 软件经验。
>
> 目标：基于 **Orange Pi Zero 2** 与 **Orange Pi 5 Max**，建立从 BootROM、U-Boot、Linux 内核、设备树、根文件系统到外设驱动和可交付镜像的完整实践能力。计划以每周 **10～15 小时**计，建议用 **18 周**完成，总投入约 **180～270 小时**。其中 10 小时/周适合按计划稳步推进，额外时间优先留给串口与硬件诊断、构建失败分析、源码阅读和结题项目验证，而不是继续压缩周期。

## 1. 先明确学习边界

这两块板都已经有可工作的 BSP、U-Boot 和内核支持。因此本计划中的“移植”不是从一张无任何软件支持的裸板开始编写 DDR 初始化，而是完成以下工作：

1. 读懂一套可启动镜像的组成和启动链路。
2. 从固定版本的源码构建 U-Boot、内核、DTB 与根文件系统。
3. 为板级硬件调整设备树、Kconfig 和启动参数。
4. 编写、构建、调试 GPIO、I2C 和 SPI 外设驱动。
5. 把这些改动固化成可复现构建和可恢复烧录流程。

真正从零支持新板还会涉及原理图、PCB、PMIC、时钟、DDR 拓扑和 DDR training。尤其 RK3588 的早期启动常依赖厂商二进制 DDR 初始化组件。这部分应在掌握本计划后，以“对照参考板做增量适配”的方式学习，不应作为第一个目标。

## 2. 两块板的分工

| 开发板 | SoC 与特点 | 本计划中的定位 |
| --- | --- | --- |
| Orange Pi Zero 2 | Allwinner H616，4 核 Cortex-A53，1 GB DDR3；MicroSD、2 MB SPI Flash、调试 UART；26 针引出 I2C、SPI、UART 与 GPIO。 | **第一主板**。硬件简单，适合先理解启动链、设备树、内核裁剪和低速外设。 |
| Orange Pi 5 Max | Rockchip RK3588，4 核 Cortex-A76 + 4 核 Cortex-A55；MicroSD、eMMC、QSPI NOR、NVMe；40 针引出 UART、I2C、SPI、CAN、PWM、GPIO，并有 MaskROM 按键。 | **第二主板**。将已掌握的方法迁移到更复杂的存储、PCIe、CAN 和多核平台。 |

先在 Zero 2 上完成最小系统和三类外设驱动，再转向 5 Max。不要一开始追逐 RK3588 的 NPU、GPU、8K 视频和完整桌面，这些会掩盖启动与驱动的核心问题。

## 3. 开始前的硬件、主机和安全准备

### 3.1 必需物料

- 一台 Linux 开发主机。Ubuntu LTS 或 Debian 均可；建议 16 GB 以上内存、100 GB 以上空闲 SSD。
- **3.3 V TTL** USB 转串口模块，以及 3 根杜邦线。必须支持 3.3 V 逻辑电平。
- 两张可靠的 MicroSD 卡，建议每块板至少一张 16 GB 或以上的专用卡。
- 稳定电源：Zero 2 使用符合官方要求的 Type-C 电源；5 Max 使用 5 V / 5 A Type-C 电源。
- 面包板、杜邦线、LED、220 Ω～1 kΩ 限流电阻、按键。
- 一个 I2C 传感器模块，例如 BME280、BH1750 或 MCP9808。
- 一个 SPI 外设，例如 MCP3008 ADC，或带控制器的数据采集模块。
- USB 逻辑分析仪。它对排查 SPI 时序、I2C NACK、UART 波特率问题非常有价值。
- 可选：隔离型 USB-RS485、CAN 收发器、两端各 120 Ω 终端电阻，用于工业现场接口练习。

### 3.2 串口接线与上电纪律

1. USB-TTL 的 **GND 接板子 GND，转接器 TX 接板子 RX，转接器 RX 接板子 TX**。
2. 不接 USB-TTL 的 5 V 或 3.3 V 供电脚。开发板只使用自己的电源适配器。
3. 先确认引脚图和丝印，再接线。不要仅按排针位置猜测 UART。
4. 串口常用设置为 `115200 8N1`，但应以该板用户手册和实际 U-Boot 日志为准。
5. 初期只从 MicroSD 启动。确认 SD 卡启动、串口日志和恢复方法都可靠以前，不修改 SPI Flash、eMMC 或 QSPI NOR。
6. GPIO 只使用 3.3 V 逻辑。LED 必须串联限流电阻；不得将 GPIO 直接接到 5 V、继电器线圈、RS485/CAN 总线或未知电平的设备。

### 3.3 建立实验仓库

在个人 Git 仓库中保存实验记录和可复现配置，源码仓库保持原样。推荐目录如下：

```text
embedded-lab/
├── README.md                 # 当前实验状态和恢复步骤
├── docs/
│   ├── boot-logs/            # 原始串口日志，按日期和板子保存
│   ├── wiring/               # 接线图、照片、引脚核对表
│   └── decisions.md          # 版本、设计取舍和故障复盘
├── manifests/
│   └── sources.lock          # U-Boot、kernel、Buildroot 的 tag/commit 与校验值
├── patches/
│   ├── u-boot/
│   └── linux/
├── dts/
├── configs/
│   ├── kernel/
│   ├── u-boot/
│   └── buildroot/
├── rootfs-overlay/
└── scripts/
    ├── build.sh
    ├── flash-sd.sh
    └── collect-boot-info.sh
```

每次开始新的可启动实验前，提交当前状态。每个构建文件和日志都要可追溯到明确的 Git commit、工具链版本和烧录镜像校验值。

## 4. 核心理论地图

先把下列链路画在纸上，并在后续每一阶段补全文件名、地址和日志位置：

```text
上电 / 复位
  → SoC BootROM
  → SPL / TPL（DRAM 初始化，某些平台含厂商组件）
  → U-Boot proper
  → kernel Image + DTB（或 FIT 中的内核与 DTB）
  → initramfs 或根文件系统
  → PID 1（BusyBox init / systemd）
  → 业务服务与外设采集程序
```

需要建立的概念包括：

- **交叉编译**：主机是 x86_64，目标是 aarch64。`aarch64-linux-gnu-` 是 GNU 工具链常见前缀；不要把主机编译出的 x86 程序复制到板子上执行。
- **设备树（DTS/DTB）**：描述板级硬件连接，不是驱动代码。内核驱动通过 `compatible`、GPIO、reg、interrupts、clocks、pinctrl 等属性与节点绑定。
- **驱动模型**：bus、device、driver、probe、deferred probe、`devm_*` 资源管理、sysfs、udev 和模块自动加载。
- **并发上下文**：进程上下文、硬中断、线程化中断、workqueue、timer。中断上下文不能睡眠。
- **SCADA 对应关系**：现场数据采集对应 IIO/input/字符设备；告警与事件对应中断、时间戳和日志；远程升级对应 A/B 分区、签名、回滚和可观测性。工业控制尤其要处理掉电、通信中断、异常恢复和版本可追溯性。

## 5. 阶段总览与验收门槛

| 阶段 | 周数 | 主板 | 主要产出 | 通过标准 |
| --- | ---: | --- | --- | --- |
| 0. 实验基线 | 1 | Zero 2 | 串口日志、主机工具链、板级资料索引 | 能反复烧录、串口登录、恢复到基线镜像。 |
| 1. 启动链观察 | 2～3 | Zero 2 | 带注释的启动时序图 | 能解释每段日志来自哪一级程序。 |
| 2. U-Boot | 4～5 | Zero 2 | 自编 U-Boot、环境与启动实验 | 能从 SD 启动自编 U-Boot 并手动加载内核。 |
| 3. 内核、设备树与裁剪 | 6～8 | Zero 2 | 自编内核、DTB、配置差异 | 能改 DT，启停外设并解释对应日志。 |
| 4. 最小根文件系统 | 9～10 | Zero 2 | Buildroot 外部树、最小镜像 | 可构建 BusyBox 系统并运行自有程序。 |
| 5. 驱动开发 | 11～14 | Zero 2 | GPIO、I2C、SPI 三个实验 | 每个实验有 DTS、驱动、测试和故障记录。 |
| 6. 工业化能力 | 15～16 | Zero 2 | 采集服务、日志、测试策略 | 断电重启后可自动恢复采集服务。 |
| 7. RK3588 迁移 | 17 | 5 Max | RK3588 启动和存储实验 | 复用方法构建、启动并调试 5 Max。 |
| 8. 结题项目 | 18 | 两板择一 | 可复现边缘采集网关镜像 | 干净主机可按文档重新构建和验证。 |

以下阶段必须按顺序完成。若一个验收条件未满足，先保留日志和最小复现，再继续排查；不要用“重新烧官方镜像”掩盖问题。

## 6. 详细周计划

### 阶段 0：实验基线和开发环境（第 1 周）

#### 理论补充

- ARMv8-A、EL0/EL1 的基本分层，用户态与内核态。
- Linux 文件系统层次、块设备、分区、挂载、串口终端。
- 交叉编译中的 build/host/target 三个概念。注意它们与 GNU Autotools 的术语在细节上不完全一致。

#### 具体操作

1. 从 Orange Pi 官方下载页获取两块板的用户手册、原理图（如提供）、官方镜像和源码说明。把 PDF 版本与下载日期记录在 `docs/decisions.md`。
2. 给 Zero 2 写入厂商官方或社区维护的稳定 Linux 镜像。烧录前先用 `lsblk -o NAME,SIZE,MODEL,TRAN` 确认目标盘；目标盘必须是 SD 卡而非主机系统盘。
3. 连接串口并完整保存从上电到登录提示符的日志：

   ```bash
   picocom -b 115200 /dev/ttyUSB0 |& tee zero2-baseline-boot.log
   ```

   串口设备可能是 `/dev/ttyUSB0` 或 `/dev/ttyACM0`，以实际枚举结果为准。
4. 在板子上记录基线：

   ```bash
   uname -a
   cat /proc/cmdline
   cat /proc/device-tree/model
   lsblk
   dmesg -T | tee dmesg-baseline.txt
   ```

5. 在开发主机安装基础工具。以 Debian/Ubuntu 为例：

   ```bash
   sudo apt update
   sudo apt install --yes \
     git build-essential bc bison flex libssl-dev libncurses-dev \
     device-tree-compiler swig python3 python3-venv \
     gcc-aarch64-linux-gnu g++-aarch64-linux-gnu \
     ccache rsync bmap-tools u-boot-tools \
     minicom picocom screen gdb-multiarch
   ```

6. 编译一个静态或动态的最小 aarch64 程序，在 Zero 2 上验证：

   ```bash
   aarch64-linux-gnu-gcc -Wall -Wextra -O2 hello.c -o hello-aarch64
   file hello-aarch64
   scp hello-aarch64 orangepi@BOARD_IP:/tmp/
   ssh orangepi@BOARD_IP /tmp/hello-aarch64
   ```

#### 本阶段产出和验收

- 一份可用的串口接线图和实拍照片。
- `zero2-baseline-boot.log`、`dmesg-baseline.txt`、镜像 SHA-256 和主机工具版本。
- 已验证的交叉编译 `hello-aarch64`。
- 从新 SD 卡启动失败时，能靠另一张基线 SD 卡恢复。

### 阶段 1：观察和拆解启动链（第 2～3 周）

#### 理论补充

- Allwinner 启动概念：BootROM 在固定启动介质位置寻找启动镜像，SPL 完成 DRAM 初始化，再进入 U-Boot。
- Rockchip 启动概念：BootROM、TPL/SPL 或 `idbloader`、U-Boot proper、内核。不同 BSP 的封装名称和偏移可能不同。
- 为什么内核与 DTB 必须匹配；为什么根文件系统中必须有可执行的 `/init` 或 `/sbin/init`。

#### 具体操作

1. 阅读 Zero 2 串口日志，为每一行打标签：BootROM 无日志或极少日志、SPL、U-Boot、内核解压、设备树解析、驱动 probe、init、登录服务。
2. 在运行中的系统中找出实际使用的启动文件：

   ```bash
   cat /proc/cmdline
   find /boot -maxdepth 3 -type f -printf '%p\n' | sort
   fw_printenv 2>/dev/null || true
   ```

3. 在 U-Boot 倒计时期间按键进入命令行，只做只读观察：

   ```text
   version
   printenv
   bdinfo
   mmc list
   part list mmc 0
   printenv bootcmd
   ```

4. 观察内核和 DTB 的加载地址、根文件系统参数、`bootargs` 与 `bootcmd`。把这些信息回填到启动时序图。
5. 使用 `fdisk -l <sd-image>` 或 `parted` 检查镜像分区布局。不要猜测裸镜像偏移；偏移由该 BSP 的镜像格式决定。

#### 验收

在一张 A4 或 Markdown 图中解释：Zero 2 的启动介质、各阶段镜像名称、U-Boot 的 `bootcmd`、kernel/DTB/rootfs 位置，以及从开机到 `PID 1` 的顺序。

### 阶段 2：在 Zero 2 上构建和调试 U-Boot（第 4～5 周）

#### 理论补充

- SPL 与 U-Boot proper 的职责差异。
- U-Boot 的 Kconfig、defconfig、环境变量、`bootm` / `booti`、extlinux 与 FIT image。
- 安全原则：U-Boot 环境保存位置和镜像偏移是板级相关数据。没有验证前不执行写 SPI/eMMC 的命令。

#### 具体操作

1. 选择一个固定的 U-Boot tag 或经确认的 BSP commit，记录到 `manifests/sources.lock`。不要用浮动的 `master` 当作实验基线。
2. 先确认树中已有的板级配置和设备树名称，而不是假设名称：

   ```bash
   find configs -iname '*zero*2*' -o -iname '*h616*'
   find arch/arm/dts -iname '*zero*2*' -o -iname '*h616*'
   ```

3. 用独立输出目录交叉构建。以下命令中的 `<board_defconfig>` 必须替换成上一步查到并与源码版本匹配的名称：

   ```bash
   export ARCH=arm
   export CROSS_COMPILE=aarch64-linux-gnu-
   make O=out/zero2 <board_defconfig>
   make O=out/zero2 -j"$(nproc)"
   ```

   某些 U-Boot 树的 64 位 Allwinner 配置使用 `ARCH=arm`，这是 U-Boot 源码树组织方式导致的；以对应 defconfig 的构建说明和实际结果为准。
4. 对照基线镜像，确定 SPL 与 U-Boot proper 的真实打包方式和介质偏移。只向**专用实验 SD 卡**写入已确认的启动组件。
5. 成功从 SD 启动自编 U-Boot 后，先在 U-Boot 命令行手动加载已知可用的内核和 DTB，再执行 `booti`。手动启动成功后，再调整 `bootcmd` 或 extlinux 配置。
6. 做一个低风险 U-Boot 修改，例如修改启动横幅或增加一个只读环境变量默认值。以 `git format-patch` 保存补丁。

#### 验收

- 串口中显示你构建的 U-Boot 版本或自定义横幅。
- 能从自编 U-Boot 启动现有的已知可用内核与根文件系统。
- 文档包含镜像组成、写入偏移、验证命令和恢复步骤。

### 阶段 3：Linux 内核、设备树与内核裁剪（第 6～8 周）

#### 理论补充

- 内核源码目录：`arch/arm64/boot/dts/`、`drivers/`、`Documentation/`、`Kconfig`、`Makefile`。
- `Image`、`modules`、`dtbs` 的用途；模块版本与 `uname -r` 的关系。
- Kconfig 依赖、`defconfig`、配置片段与最小差异配置。
- DTS 的节点继承、label/phandle、`status = "okay"`、pinctrl、GPIO flags、`interrupt-parent`。

#### 具体操作

1. 先选择一个路径：
   - **主线路径**：优先用于理解标准 Linux 结构和便于阅读的驱动代码。
   - **厂商 BSP 路径**：当某硬件在主线还不能工作时使用。把厂商补丁与主线差异单独保存，不混入无关修改。

2. 固定 kernel tag/commit 后，以独立目录构建：

   ```bash
   export ARCH=arm64
   export CROSS_COMPILE=aarch64-linux-gnu-
   make O=out/zero2 <known_good_defconfig>
   make O=out/zero2 -j"$(nproc)" Image modules dtbs
   make O=out/zero2 INSTALL_MOD_PATH="$PWD/staging/zero2" modules_install
   ```

   `O=` 让源码目录保持干净；Linux Kbuild 官方文档明确支持以 `O=` 或 `KBUILD_OUTPUT` 指定输出目录。

3. 用实际源码树确认 Zero 2 的 DTS/DTB 文件名：

   ```bash
   find arch/arm64/boot/dts -iname '*orangepi*zero*2*' -o -iname '*h616*'
   ```

4. 先只做一个设备树可见的、低风险的修改，例如启用状态 LED，或为一个尚未启用的 I2C 控制器添加 `status = "okay"`。必须同时补齐 pinctrl，不能只改 `status`。
5. 复制内核 `Image`、正确 DTB、模块到专用实验 SD 卡。保留基线内核条目，使用 extlinux 的额外启动项或不同文件名实现回退。
6. 启动后验证：

   ```bash
   uname -a
   dmesg -T | less
   cat /proc/device-tree/model
   zcat /proc/config.gz > running.config  # 如果内核启用了 IKCONFIG_PROC
   ```

7. 使用 `scripts/diffconfig` 或版本控制对比配置，只保留为实验所需的差异。每启用一个选项，都写清依赖和原因。

#### 验收

自编内核和自编 DTB 启动成功，串口日志中能确认修改后的设备节点或驱动完成 probe；出现失败时，可通过基线启动项恢复。

### 阶段 4：用 Buildroot 构建最小系统（第 9～10 周）

#### 为什么先学 Buildroot

Buildroot 能在一次构建中生成交叉工具链、根文件系统、内核和引导加载器，适合学习“配置决定整个镜像”的关系。它比 Yocto 的元数据模型简单，特别适合作为第一个可复现镜像工程。掌握 Buildroot 后，再学习 Yocto/OpenEmbedded 的 layer、recipe 和 sstate 缓存。

#### 理论补充

- BusyBox、init、`/etc/inittab`、getty、devtmpfs、mdev/udev。
- rootfs overlay、post-build script、post-image script 的不同职责。
- 为什么不应直接修改 Buildroot 的 `output/target`：`make clean` 后改动会丢失。

#### 具体操作

1. 固定 Buildroot 版本。选择发布版本或确定 commit，记录到 `sources.lock`。
2. 查找是否已有相关 defconfig；没有时从 aarch64 通用最小配置开始：

   ```bash
   make list-defconfigs | grep -Ei 'orange|h616|rockchip|rk3588'
   make O="$PWD/out/zero2" menuconfig
   ```

3. 配置最小系统：aarch64、BusyBox init、串口 getty、以太网或 USB 网络、`dropbear` SSH、`i2c-tools`、`libgpiod` 工具、`strace`、`gdbserver`。调试期保持足够工具，产品配置再裁剪。
4. 不把自定义文件直接写入 `output/target`。建立 `br2-external`：

   ```text
   br2-external/
   ├── external.desc
   ├── Config.in
   ├── external.mk
   ├── configs/
   │   └── zero2_lab_defconfig
   ├── board/zero2/
   │   ├── kernel.config
   │   ├── rootfs-overlay/
   │   ├── post-build.sh
   │   └── post-image.sh
   └── package/scada-collector/
       ├── Config.in
       └── scada-collector.mk
   ```

5. 用 `BR2_EXTERNAL` 和 `O=` 构建，保证 Buildroot 源码、定制目录和构建输出相互独立：

   ```bash
   make -C buildroot \
     BR2_EXTERNAL="$PWD/br2-external" \
     O="$PWD/out/zero2" zero2_lab_defconfig
   make -C buildroot O="$PWD/out/zero2"
   ```

6. 在 overlay 中加入 `/etc/motd`、网络配置、一个只读版本文件和一个最小启动脚本。编写 `scada-collector` 的 C/C++ 程序，交叉编译并由 Buildroot 打包。
7. 在全新输出目录进行第二次构建，验证镜像没有依赖手工修改。

#### 验收

- Zero 2 从 Buildroot 镜像启动到串口登录。
- `scada-collector --version` 可运行，版本包含 Git commit 或构建 ID。
- 删除 `out/zero2` 后，按 README 仍能重新生成等价镜像。

### 阶段 5A：GPIO 与中断驱动（第 11 周）

#### 先做用户态验证

1. 在 DTS 中为 LED 和按键使用正确的 GPIO controller、pinmux 与电气 flags。
2. 使用现代 character-device GPIO 接口和 `libgpiod` 工具观察线路。不要学习已废弃的 sysfs GPIO 导出接口。
3. LED 只用于输出；按键配置为输入并适当上拉/下拉。用万用表核对空闲电平与按下电平。

#### 驱动练习

编写一个 `lab,scada-dio` 平台驱动，并用设备树节点绑定。驱动第一版只完成：

- 在 `probe()` 中用 `devm_gpiod_get()` 获取 `alarm` 输出和 `input` 输入。
- 为输入申请线程化 IRQ，设置合适的触发边沿。
- 在中断上半部只做最少工作；耗时处理放入线程化 handler 或 workqueue。
- 用 `miscdevice` 或清晰的 sysfs 属性向用户态报告输入状态和计数。
- 使用 `dev_err_probe()` 报告资源未就绪，理解 deferred probe。

#### 验收

按键触发后，驱动的事件计数递增；LED 按用户态命令或策略切换。提交内容必须包括 DTS、驱动、Kconfig/Makefile、加载日志、接线图和测试步骤。

### 阶段 5B：I2C 传感器与设备树绑定（第 12～13 周）

#### 理论补充

- I2C start/stop、地址、ACK/NACK、重复起始、时钟拉伸；open-drain 与上拉电阻。
- Linux I2C adapter 与 I2C client；`reg` 是从设备地址，不是内存地址。
- `regmap`、runtime PM、IIO subsystem 的适用场景。

#### 具体操作

1. 先在已知安全的 I2C 控制器和已连接传感器上做用户态验证：

   ```bash
   i2cdetect -l
   i2cdetect -y <bus-number>
   i2cget -y <bus-number> <address> <register>
   ```

   不要扫描连接 PMIC、EEPROM、工业设备或未知设备的 I2C 总线。某些设备对探测命令有副作用；只操作已核实的实验总线。
2. 用逻辑分析仪确认 3.3 V 电平、SCL/SDA 时序和从机 ACK。
3. 先在 DTS 中实例化内核已有驱动支持的传感器，例如 BME280，并观察对应的 IIO sysfs 节点。阅读上游驱动与 binding 文档。
4. 第二步选择一个寄存器简单、公开数据手册的传感器，例如 BH1750 或 MCP9808，编写自己的实验性 `i2c_driver`：
   - `of_match_table` 中使用实验专用 compatible，例如 `lab,bh1750`。
   - `probe()` 验证芯片 ID 或执行无副作用的复位/初始化。
   - 先通过 `read_raw` 或只读 sysfs 属性提供单个物理量。
   - 遇到总线错误应返回标准 errno，不可吞掉错误后伪造读数。
   - 加入采样间隔、超时、断线和重复 probe 的测试。

#### 验收

传感器节点由 DTS 自动创建，无需手工 `insmod` 参数；重启后可稳定读取物理量。提交一个故意拔掉传感器时的日志，证明驱动能明确报错且系统没有卡死。

### 阶段 5C：SPI 数据采集（第 14 周）

#### 理论补充

- SPI mode 0～3、CPOL/CPHA、片选、位宽、最大时钟与全双工。
- `spi_controller`、`spi_device`、`spi_driver` 和 `spi_transfer`。
- 数据采集中的采样率、抖动、时间戳和缓冲队列；这些直接关联 SCADA 历史数据的可信度。

#### 具体操作

1. 按传感器/ADC 数据手册接 MOSI、MISO、SCLK、CS、GND 和电源。先用逻辑分析仪确认片选和 SPI mode。
2. 在受控原型阶段可使用 `spidev` 验证链路，但不要把“把任意设备强绑定为 `spidev`”当作正式设备树方案。
3. 阅读内核中现有 ADC 或同类传感器的 SPI 驱动。选择 MCP3008 时重点学习其 IIO 驱动、通道描述和设备树 binding。
4. 编写一个只支持当前实验硬件的最小 SPI 驱动或扩展现有实验驱动：
   - 在 `probe()` 中初始化 SPI mode、word length 和最大频率。
   - 用明确的帧格式读取一帧数据，检查长度和返回值。
   - 使用 IIO 或字符设备把数据提供给用户态。
   - 测试 1 Hz、10 Hz、100 Hz 的采样，记录平均周期和最大抖动。
5. 用户态 `scada-collector` 读取数据，在本地以带单调时间戳的 CSV/二进制日志保存。断网时本地缓存，网络恢复后再上传是后续工业网关的基本模式。

#### 验收

- 正确读取至少一个 SPI 通道的可重复数据。
- 逻辑分析仪抓包与驱动配置的 mode/时钟一致。
- 文档说明采样时间戳来自哪里，以及网络异常时数据如何处理。

### 阶段 6：面向工业场景的系统能力（第 15～16 周）

你的 SCADA 经验在这一阶段最有优势。目标不是堆积功能，而是让采集程序在不可靠的现场条件下可诊断、可恢复。

#### 建议课题

1. **服务管理**：为 `scada-collector` 编写 BusyBox init 脚本或 systemd unit。定义启动依赖、退出码和自动重启策略。
2. **时间**：区分 `CLOCK_MONOTONIC` 与实时时钟。采样间隔和性能度量使用单调时钟；业务事件显示使用 UTC/本地时间，并处理 NTP 尚未同步的情况。
3. **看门狗**：研究硬件 watchdog 与 systemd watchdog。先做软件层健康检查，确认语义后才启用真正会复位系统的 watchdog。
4. **日志**：用 log level、日志轮转和启动原因记录。记录内核版本、DTB 哈希、根文件系统版本、应用版本和配置版本。
5. **故障注入**：拔掉 I2C 传感器、断开以太网、重启服务、反复冷启动、填满日志分区。每种故障都必须有预期行为和恢复验证。
6. **RS485/CAN（可选）**：使用隔离收发器。RS485 注意 A/B 标识不统一的问题；CAN 只在总线两端装 120 Ω 终端电阻。先使用 `can-utils` 观察 SocketCAN，再考虑协议栈。

#### 内核调试工具清单

```bash
# 驱动绑定与设备树信息
ls /sys/bus/i2c/devices/
ls /sys/bus/spi/devices/
find /sys/firmware/devicetree/base -maxdepth 3 -type f

# 日志与动态调试
dmesg -wH
cat /sys/kernel/debug/gpio 2>/dev/null

# 模块信息
modinfo <module>
lsmod

# 应用和系统调用
strace -f -tt -o collector.strace ./scada-collector
gdbserver :2345 ./scada-collector
```

使用 ftrace、trace-cmd、perf 前先明确问题，例如“IRQ 到用户态事件延迟是多少”。避免先收集大量无目标的 trace。

#### 验收

完成一个故障矩阵。至少覆盖：传感器不存在、I2C NACK、网络中断、进程异常退出、系统冷启动。每项包含触发方法、日志、预期结果、实际结果和修复提交。

### 阶段 7：把方法迁移到 Orange Pi 5 Max（第 17 周）

#### 目标

5 Max 的价值不在于重复 Zero 2 的 GPIO 点灯，而在于认识复杂 SoC 和更多启动/存储介质如何改变工程流程。

#### 具体操作

1. 重做阶段 0：独立串口日志、独立基线镜像、独立接线图和恢复步骤。5 Max 有 MaskROM 按键，但进入恢复模式、主机工具和烧录流程必须以当前官方手册/BSP 为准。
2. 从 SD 卡启动已知可用镜像。确认 SD、eMMC、QSPI NOR 和 NVMe 的实际检测顺序与启动优先级；不要基于其他 Orange Pi 型号的教程操作。
3. 从当前 BSP 中确认 RK3588 的启动组件：BootROM、`idbloader`/SPL、U-Boot、可能的 trust/firmware 文件、内核与 DTB。画出与 H616 的差异。
4. 构建和启动一个最小内核或 Buildroot 系统。先保证串口和以太网，再启用 NVMe 和一个低速外设。
5. 在 40 针接口上迁移已完成的一个 I2C 或 SPI 实验。必须重新核对引脚复用和电压，不能照搬 Zero 2 的物理引脚号。
6. 练习一个 5 Max 专属课题：
   - NVMe 根文件系统或数据分区；或
   - SocketCAN 收发测试；或
   - PCIe 枚举与设备驱动日志分析。

#### 验收

5 Max 能从你的可复现构建产物启动；串口日志、存储布局、启动组件和一个已迁移外设均有文档。能恢复到官方基线镜像。

### 阶段 8：结题项目，构建一个小型边缘采集网关（第 18 周）

选择 Zero 2 或 5 Max，建议先在 Zero 2 交付，再把同一项目迁移到 5 Max。

#### 最小功能要求

- Buildroot 生成的只读或受控可写根文件系统。
- U-Boot、内核、DTB、rootfs 和应用版本均固定并可追溯。
- 一个 GPIO 数字输入、一个 I2C 传感器、一个 SPI 采集通道。
- `scada-collector` 将带时间戳的数据写本地环形文件，并通过 MQTT/HTTP/自定义 TCP 之一上报。
- 网络断开时持续本地缓存，恢复后按顺序补传。
- 设备启动自检、健康状态、日志轮转、故障码。
- 一键构建脚本、SD 卡烧录说明、串口恢复说明和验收脚本。

#### 推荐验收清单

```text
[ ] 在干净输出目录完成构建
[ ] 校验镜像 SHA-256
[ ] 写入专用 SD 卡并从串口启动
[ ] GPIO、I2C、SPI 三类外设工作
[ ] 拔掉一个传感器后应用可报告明确错误
[ ] 断网 10 分钟后数据保存在本地，恢复后补传
[ ] 连续冷启动 20 次无人工干预
[ ] 通过基线 SD 卡或官方恢复工具回退
[ ] README 足以由另一位开发者复现实验
```

## 7. 推荐学习顺序，而不是资料堆砌

### 必读的上游文档

- [U-Boot Build U-Boot 文档](https://docs.u-boot.org/en/latest/build/index.html)：源码获取、交叉构建、Kconfig 和可复现构建。
- [Linux Kernel Kbuild 文档](https://docs.kernel.org/kbuild/kbuild.html)：`ARCH`、`CROSS_COMPILE`、`O=`/`KBUILD_OUTPUT`、模块和 DTB 安装路径。
- [Linux DeviceTree 文档](https://docs.kernel.org/devicetree/index.html)：设备树模型、binding 和维护规范。
- [Linux IIO 子系统文档](https://docs.kernel.org/driver-api/iio/index.html)：传感器、ADC 和缓冲采集驱动。
- [Linux GPIO 用户态接口文档](https://docs.kernel.org/userspace-api/gpio/index.html)：基于 character device 的 GPIO API。
- [Buildroot 手册](https://buildroot.org/downloads/manual/manual.html)：交叉工具链、根文件系统、`br2-external`、overlay、包和镜像构建。
- [Yocto Project Quick Build 文档](https://docs.yoctoproject.org/dev/brief-yoctoprojectqs/index.html)：在结题项目完成后再进入 Yocto。

### 板级资料

- [Orange Pi Zero 2 官方 Wiki](http://www.orangepi.org/orangepiwiki/index.php/Orange_Pi_Zero_2)：确认 H616、串口、26 针接口和电源要求。
- Orange Pi 5 Max 官方下载页、用户手册、原理图与对应 BSP：以下载时的版本为准，记录本地文件哈希和发布日期。
- 所用传感器、ADC、收发器的数据手册：驱动开发时它比网络教程更重要。

### 阅读源码的方法

对每个正在使用的内核驱动，按这个顺序阅读：

1. 找到对应的 YAML binding，确认 DTS 属性语义。
2. 阅读 `of_match_table`、`probe()` 和错误处理路径。
3. 跟踪数据进入哪个 subsystem，例如 IIO、input、hwmon、netdev 或 miscdevice。
4. 查看 Kconfig 依赖和 Makefile 目标。
5. 用 `git log -- <file>` 了解关键设计决策和修复历史。

## 8. 调试方法和常见陷阱

| 现象 | 优先检查 | 不要做 |
| --- | --- | --- |
| 无串口输出 | 3.3 V 电平、GND、TX/RX 交叉、正确 UART、波特率、电源。 | 反复刷写 eMMC/SPI，或接入 USB-TTL 的供电脚。 |
| U-Boot 有输出但内核不启动 | `bootargs`、`Image` 地址、DTB 是否匹配、根文件系统参数。 | 随机替换多个组件，导致失去最小复现。 |
| `Waiting for root device` | 分区 UUID、驱动是否内建、`root=` 参数、文件系统支持。 | 只把模块复制到 rootfs，却没有可加载的模块依赖。 |
| 驱动未 probe | DTS `compatible`、节点 status、pinctrl、Kconfig、供电/时钟/复位资源。 | 先修改驱动逻辑而不检查 `dmesg` 和 `/sys/bus/.../devices`。 |
| I2C 全是 NACK | SDA/SCL 接反、无共地、错误电平、缺少上拉、错误总线或地址。 | 在未知总线持续全地址扫描。 |
| SPI 数据固定为 0/全 1 | MISO/MOSI 接反、CS 不对、mode 错、时钟过快、外设未供电。 | 盲目提高 SPI 频率。 |
| Buildroot 改动丢失 | 是否改了 `output/target`。 | 把生成目录当作源代码仓库。 |

每次故障都按以下模板记录：**目标、最小复现、版本、接线、完整串口日志、预期、实际、已排除项、最终原因、修复 commit**。这是从“会点灯”走向“能维护 BSP”的关键习惯。

## 9. 后续路线

完成结题项目后，可按职业目标分支深入：

- **BSP/驱动方向**：kernel upstream workflow、DT binding YAML、DMA、pinctrl、regulator、clock、PM、PCIe、网络驱动和 PREEMPT_RT。
- **产品发行版方向**：Yocto/OpenEmbedded、SWUpdate 或 RAUC、A/B 更新、SBOM、CVE 跟踪、Secure Boot、可信启动链。
- **工业边缘方向**：CANopen、Modbus RTU/TCP、OPC UA、MQTT Sparkplug、PTP、TSN、IEC 62443 的设备加固与资产管理。
- **性能与可靠性方向**：ftrace、perf、eBPF、cyclictest、锁竞争、内存泄漏、掉电一致性和故障注入。

## 10. 完成标准

达到下列标准，才算真正完成本学习计划：

1. 你能看串口日志定位故障发生在 BootROM/SPL/U-Boot/kernel/init 的哪一层。
2. 你能从固定源码、配置与补丁构建可启动镜像，而非依赖手工修改的 SD 卡。
3. 你能为一个真实 GPIO、I2C 或 SPI 外设写出 DTS 和最小驱动，并说明资源、并发和错误路径。
4. 你能在两块不同 SoC 的板子上重复上述方法，并解释启动组件和恢复方式的差异。
5. 你的镜像、应用、配置、日志和烧录/恢复步骤可由其他开发者复现。

这套能力与工业 SCADA 的系统经验结合后，可以自然延伸到工业网关、协议转换、现场数据采集和设备 BSP 维护工作。
