# MCP2518FD SPI-to-CAN 驱动移植与问题排查（MIC-712AR / Orin NX）

> 平台：Advantech MIC-712AR + Jetson Orin NX，L4T R36.4.3 / JetPack 6.2，内核 `5.15.148-tegra`
> 芯片：**Microchip MCP2518FD**（SPI 转 CAN FD 控制器），挂在 **SPI1（spi0.0）**，40 MHz 晶振
> 链路：1 Mbit/s 仲裁 + 5 Mbit/s 数据（CAN FD），`nINT` → **AON GPIO PEE.02**
> 业务需求：**稳定 2 kHz（500 µs/帧）收发，零丢帧**
> 结论：**已达成**（中断模式，空闲 CPU ~0.6%）

---

## 1. 硬件与拓扑

```
Jetson Orin NX ── SPI1(spi@3210000, CS0, 10MHz) ── MCP2518FD ── CAN FD ── 上位机/DUT
                        │                              │
                        └── IRQ: AON GPIO PEE.02 ◄─────┘ nINT（开漏，低有效，LEVEL）
```

| 项 | 值 | 依据 |
|---|---|---|
| SPI 控制器 | `spi@3210000`（SPI1），CS0 | DT `live.dts` |
| 芯片节点 | `spi@3210000/can@0`，`compatible = "microchip,mcp2518fd"` | DT |
| 振荡器 | `mcp2518fd-oscillator` = 40 MHz fixed-clock | DT |
| 中断 | `interrupts-extended = <&gpio_aon PEE.02 IRQ_TYPE_LEVEL_LOW>` | DT |
| GPIO 控制器 | `gpio@c2f0000`，`nvidia,tegra234-gpio-aon`（4 条 GIC SPI 56..59） | DT |
| 位定时 | `bitrate 1000000 / dbitrate 5000000`，FD | `ip -details` |

---

## 2. 驱动移植

### 2.1 驱动本体（内核树内，无需外部源码）

```
drivers/net/can/spi/mcp251xfd/
├── mcp251xfd-core.c      主逻辑（probe / 中断 / TX / RX / 我们加的 poll 与参数）
├── mcp251xfd.h           结构体
├── mcp251xfd-ring.c      TX/RX FIFO 环
├── mcp251xfd-regmap.c    SPI regmap（含 CRC）
├── mcp251xfd-crc16.c
└── mcp251xfd-dump.c      debugfs dump
```

内核配置需要（`CONFIG_CAN_MCP251XFD=m`，依赖 `CONFIG_CAN`、`CONFIG_SPI`、`CONFIG_CAN_MCP251XFD_SYSCON=n`）。

### 2.2 设备树（两份改动）

**(a) 板级 DTSI**（`custom/kernel/.../nv-platform/tegra234-mic712ar-mcp2518fd.dtsi`）：

```dts
/ {
	mcp2518fd_osc: mcp2518fd-oscillator {
		compatible = "fixed-clock";
		#clock-cells = <0>;
		clock-frequency = <40000000>;
	};

	bus@0 {
		spi@3210000 {
			/delete-node/ spi@0;          /* 默认的 spidev 节点要去掉 */
			can@0 {
				compatible = "microchip,mcp2518fd";
				reg = <0>;
				clocks = <&mcp2518fd_osc>;
				spi-max-frequency = <10000000>;   /* 实测上限见 §4.3 */
				interrupts-extended = <&gpio_aon
					TEGRA234_AON_GPIO(EE, 2)
					IRQ_TYPE_LEVEL_LOW>;
				interrupt-gpios = <&gpio_aon
					TEGRA234_AON_GPIO(EE, 2)
					GPIO_ACTIVE_LOW>;         /* 可选：门控轮询用 */
			};
		};
	};
};
```

**(b) MB1 BCT（关键！）** `custom/bsp/bootloader/tegra234-mb1-bct-gpioint-p3767-0000.dts`：

```dts
port@EE {
    pin-0-int-line = <4>;
    pin-1-int-line = <4>;
    pin-2-int-line = <4>;    /* ★ 必须有！否则 PEE.02 永远不产生中断 */
    pin-3-int-line = <4>;
    ...
};
```

### 2.3 模块参数（`/etc/modprobe.d/`）

```
# 推荐默认：纯中断
options mcp251xfd poll_us=0

# 中断不可用时的兜底（GPIO 电平门控轮询）
options mcp251xfd poll_us=200 gpio_poll=1
```

| 参数 | 作用 |
|---|---|
| `poll_us` | 0=纯中断；N=每 N µs 轮询一次控制器（hrtimer 驱动，不受 `HZ=250` 限制） |
| `gpio_poll` | 1=轮询时用 INT 电平门控：只有 INT assert 才做 SPI 排空（空闲几乎零成本） |
| `spi_hz` | 覆盖 SPI 时钟（调参用，0=用 DT 值） |
| `irq_flags_mode` | 0=DT 触发类型，1=low，2=falling，3=rising，4=both（排查用） |

### 2.4 板级初始化脚本

```bash
ip link set can1 type can bitrate 1000000 dbitrate 5000000 fd on \
        loopback off listen-only off one-shot off berr-reporting on
ip link set can1 txqueuelen 1000        # 发送队列（默认 10 太小，2 kHz 突发会 ENOBUFS）
ip link set can1 up
```

> 注意：`ip link set type can ...` 是**按位更新**的，`loopback off` 必须显式写，否则内部回环仍然开着（表现为 RX = 2×TX、无需 ACK 也能发）。

---

## 3. 排查流程（本次真实经历，按时间线）

### 3.1 症状一：CAN 完全不通

- 现象：`ip link set can1 up` 报 `Invalid argument`；或起来了但无帧
- 原因：**没有位定时**（必须先 `type can bitrate ... dbitrate ... fd on` 再 `up`）
- 另：`dbitrate` 与 `fd on` 必须在**同一条**命令里

### 3.2 症状二：能收但不能发 / 发送极慢、`bus-errors` 暴涨

- 现象：`ip -s -d link show can1` 里 `bus-errors` 持续暴涨（本次见到 140128）
- 原因：**总线上没有其它节点 ACK**（对端没开）；CAN 会一直重试
- 教训：**任何收发测试都必须有对端在线**（或开 `loopback` 自环）

### 3.3 症状三（核心）：中断计数恒为 0 —— "引脚有信号，内核收不到"

证据链：

| 检查 | 结果 |
|---|---|
| `/proc/interrupts` 中 `c2f0000.gpio 25 Level spi0.0` | **恒为 0** ✗ |
| 同板 `Power`（PEE.04）中断 | 也恒为 0 ⇒ **AON GPIO 中断整体不投递** |
| `/sys/kernel/debug/gpio` 中该线电平 | `in hi IRQ ACTIVE LOW`（未 assert 时正确） |
| 总线有流量时采样该线电平 | **在 lo/hi 之间翻转** ⇒ **信号确实到达 SoC** ✓ |
| DT `interrupts-extended` | `<&gpio_aon PEE.02 LEVEL_LOW>` ✓ 配置正确 |
| `irq_flags_mode` 试 0/1/2/4 四种触发 | 全为 0 ✗ |
| **读取 AON GPIO security 区（0xc2f0000）** | **内核 SError → 板子重启** ⇒ 该区有防火墙保护 |
| **MB1 BCT gpioint 的 `port@EE`** | **缺 `pin-2-int-line`** ✗✗ |

**根因**：MB1 BCT 没有把 PEE.02 路由到 AON 中断线（同端口其它 pin 都有，唯独 pin-2 漏了）
⇒ 引脚有电平，但硬件从不向 GIC 发中断 ⇒ 内核 IRQ 永远不触发

**修复**：`port@EE` 补 `pin-2-int-line = <4>;` → 重新生成 BCT → **重刷** → 中断立即可用（计数随流量增长）

### 3.4 ⚠️ 最大的坑：轮询会把中断"吃掉"

MCP2518FD 的 `nINT` 是**电平**中断（有未处理标志时拉低）：

```
中断路径：INT 拉低 → GIC → 驱动 handler → 清标志 → INT 释放
轮询路径：每 N µs 主动读芯片 → 读的同时清掉标志 → INT 释放
```
⇒ **只要轮询还开着，电平中断永远看不到**（计数恒 0），会误判成"中断没修好"。
⇒ **验证中断必须 `poll_us=0`**，且判据要两个一起看：
1. `/proc/interrupts` 该行计数 **> 0 并增长**
2. `poll_us=0` 下 `cansend2k -n 2000 -r 2000` 仍能 `sent=2000`（说明中断真的在回收 TX/RX）

### 3.5 SPI 时钟上限（实测）

| 频率 | 结果 |
|---|---|
| 10 MHz | ✅ 稳定 |
| 12 MHz | ✅ 可用 |
| 15 / 16 / 17 / 20 MHz | ❌ `probe failed -110`：`Timeout waiting for Oscillator Ready` + CRC 读错 |

> >12 MHz 失败后芯片会**卡住**（连 10 MHz 也起不来）→ 用 **1 MHz 低速探针**可救回：
> `modprobe mcp251xfd spi_hz=1000000`，成功后再正常加载。

### 3.6 命令行实测手册（按症状，可直接复制）

#### 症状 A：CAN 完全不通（`ip link set can1 up` 报错 / 没有 can1）

```bash
# 1) 模块在不在、参数对不对
lsmod | grep mcp251xfd
modinfo -p mcp251xfd | head
cat /sys/module/mcp251xfd/parameters/{poll_us,spi_hz,gpio_poll}

# 2) 芯片是否初始化成功（最关键的一行）
sudo dmesg | grep -iE 'mcp251xfd|mcp2518' | tail -15
#   期望: "MCP2518FD rev0.0 (...) c:40.00MHz m:10.00MHz r:10.00MHz e:0.00MHz) successfully initialized."
#   若见 "Timeout waiting for Oscillator Ready" / "CRC read error" ⇒ SPI 通信有问题（见 §3.5）

# 3) 有没有 netdev
ls /sys/class/net/ | grep can
ip -o link show can1 2>&1

# 4) 位定时（up 失败最常见原因：还没设 bitrate）
ip -details link show can1            # 看 bitrate/dbitrate 是否为空
sudo ip link set can1 down
sudo ip link set can1 type can bitrate 1000000 dbitrate 5000000 fd on \
     loopback off berr-reporting on  # ← fd on 与 dbitrate 必须同一条命令
sudo ip link set can1 up
ip -details link show can1 | head -5  # 期望 state UP + bitrate 1000000 + dbitrate 5000000
```

#### 症状 B：能收不能发 / 发送极慢 / `bus-errors` 暴涨

```bash
# 1) 看总线错误（ACK 错误会体现在 bus-errors）
ip -s -d link show can1 | grep -A2 'bus-errors'
#   持续增长 ⇒ 总线上没有其它节点 ACK（对端没开）

# 2) 用 loopback 自环验证"芯片→驱动→协议栈"这条链本身是好的
sudo ip link set can1 down
sudo ip link set can1 type can bitrate 1000000 dbitrate 5000000 fd on loopback on berr-reporting on
sudo ip link set can1 up
candump can1 &                 # 或 canrx can1 20
sleep 0.5
sudo cansend2k -i can1 -n 100 -r 1000        # 自环下应能发出并被自己收到
```

#### 症状 C（核心）：中断计数恒为 0 —— **看 GPIO 电平变化**

**① 先看中断与 GPIO 现状**

```bash
# 中断计数（关键判据）
grep 'c2f0000.gpio' /proc/interrupts
#   268:  0 ... c2f0000.gpio  25 Level  spi0.0      ← 恒 0 = 中断没投递

# GPIO 线是否被驱动占用、当前电平、极性
cat /sys/kernel/debug/gpio | grep -i mcp
#   gpio-341 (PEE.02 | mcp2518-int) in hi IRQ ACTIVE LOW   ← ACTIVE LOW 表示低电平=有中断

# 该控制器的父中断（AON GPIO 的 4 条 GIC SPI 56..59）有没有登记
cat /proc/device-tree/bus@0/gpio@c2f0000/interrupts | od -An -tx1
cat /proc/device-tree/bus@0/gpio@c2f0000/interrupt-controller | od -An -tx1
```

**② 一边制造中断，一边采样 GPIO 电平（判断信号是否到达 SoC）**

```bash
# 采样电平的小函数（hi/lo）
level() { sudo cat /sys/kernel/debug/gpio | grep 'mcp2518-int' \
          | grep -oE '\) +(in|out) +[a-z]+' | awk '{print $NF}' | head -1; }

# 开 loopback 产生中断（芯片 eINT 会随收发 assert/deassert）
sudo ip link set can1 down
sudo ip link set can1 type can bitrate 1000000 dbitrate 5000000 fd on loopback on berr-reporting on
sudo ip link set can1 up

# 后台持续发帧
sudo cansend2k -i can1 -n 3000 -r 1000 -q &

# 采样 15 次（每 80 ms）
echo -n "level: "; for i in $(seq 1 15); do echo -n "$(level) "; sleep 0.08; done; echo

# 断言判据：
#   电平出现 lo（或序列在 hi/lo 间跳变） ⇒ 信号【到达 SoC 引脚】✓ ⇒ 问题在中断使能/路由
#   全程 hi（不动）                    ⇒ 信号【没到】✗ ⇒ 查走线/pinmux
```

**③ 判定方向后继续查**

```bash
# 若"电平在动但 IRQ 恒 0" ⇒ 查中断路由/BCT：
grep -n 'pin-2-int-line' <BSP>/vendor/bsp/bootloader/tegra234-mb1-bct-gpioint-p3767-0000.dts
#   port@EE 里必须有 pin-2-int-line = <4>;（同端口其它 pin 都有，唯独 pin-2 容易漏）

# 触发类型是否影响（0=DT,1=low,2=falling,3=rising,4=both）
sudo ip link set can1 down; sudo modprobe -r mcp251xfd
sudo modprobe mcp251xfd irq_flags_mode=2; sudo ip link set can1 up
grep 'c2f0000.gpio' /proc/interrupts

# 极端手段：直接读寄存器（危险！AON security 区读一次会 SError 重启，勿乱试）
#   安全做法：只读 gpio 区（例如 PEE.02 的 pin 寄存器 0xc2f1648），不要读 0xc2f0000
busybox devmem 0xc2f1648 32
```

> **重要**：上述电平采样必须在 `poll_us=0`（纯中断）下做一次，否则轮询会提前清掉芯片中断标志，
> 电平和 IRQ 都会表现成"没有中断"（见 §3.4）。

#### 症状 D：RX 丢帧 / `FIFO overflow`

```bash
# 1) 溢出计数（ip -s 的 errors 列就是 overflow 次数）
ip -s -d link show can1 | tail -6
sudo dmesg | grep -c 'FIFO overflow'
sudo dmesg | grep 'FIFO overflow' | tail -3

# 2) 是否有人在读 socket（没人读 ⇒ 内核队列满 ⇒ 反压驱动 ⇒ 芯片 FIFO 溢出）
pgrep -af 'candump|canrx|canplayer'

# 3) 正确对账丢帧：发送端 ID 递增，接收端看总数与连续性（candump 自己会丢，别用它判）
sudo cansend2k -i can1 -n 2000 -r 2000        # ID 0x001..0x7D0
# 接收侧用 canrx（带间隔统计）：
/tmp/canrx can1 10
```

#### 症状 E：速率/抖动不达标

```bash
# 发送侧（精确绝对时钟 + 可选 SCHED_FIFO）
sudo cansend2k -i can1 -n 20000 -r 2000 -R
#   看 actual rate / interval avg·min·max / jitter / hist

# 接收侧逐帧间隔（6 位小数直方图）
/tmp/canrx can1 10

# 注意：PC 上位机工具的时间戳常常只有 1 ms 精度，无法判断 0.5 ms 间隔
```

---

## 4. 修改点汇总

### 4.1 驱动（`mcp251xfd-core.c` / `.h`）

1. **`poll_us`**（hrtimer + workqueue 轮询）
   - 为什么用 hrtimer：`CONFIG_HZ=250` ⇒ `queue_delayed_work` 最小 4 ms，2 kHz 根本不够
   - 守护：`STOPPED/SLEEPING/BUS_OFF` 状态跳过；`netif_running()` 检查
2. **`gpio_poll`**（GPIO 电平门控）
   - hrtimer 回调里先 `gpiod_get_value()`（MMIO，~200 ns），**只有 INT assert 才 schedule_work** 做 SPI 排空
   - 效果：空闲 CPU 从 8%（盲轮询）降到 ~0.6%
3. **`spi_hz`**（运行时覆盖 SPI 时钟，便于扫上限）
4. **`irq_flags_mode`**（运行时改触发类型，排查用）
5. 有界读取：中断 handler 内单次最多处理 N 帧，避免 flag 反复 latch 导致死循环

### 4.2 设备树 / BCT

- `tegra234-mic712ar-mcp2518fd.dtsi`：`can@0` 节点 + 40 MHz fixed-clock + `interrupts-extended` + `interrupt-gpios`
- **`tegra234-mb1-bct-gpioint-p3767-0000.dts`：`port@EE { pin-2-int-line = <4>; }`** ← 本次核心修复

### 4.3 用户态

- `modprobe.d`：默认 `poll_us=0`
- systemd service（开机 `ip link set ... up`）
- 验收工具：`cansend2k`（精确速率发送）、`canrx`（逐帧间隔统计）

---

## 5. 调试手段（可复用）

| 目的 | 命令 |
|---|---|
| 中断是否在投递 | `grep 'c2f0000.gpio' /proc/interrupts`（看计数是否增长） |
| GPIO 线与电平 | `cat /sys/kernel/debug/gpio \| grep -i mcp` |
| 控制器状态/错误 | `ip -details -statistics link show can1`（`bus-errors`/`errors`/`dropped`） |
| 位定时确认 | `ip -details link show can1`（`bitrate`/`dbitrate`/`sample-point`） |
| 芯片初始化与速率 | `dmesg \| grep mcp251xfd`（`c:40.00MHz m:10.00MHz r:10.00MHz`、`successfully initialized`） |
| RX 溢出 | `dmesg \| grep 'FIFO overflow'`（`ip -s` 的 `errors` 列 = 溢出计数） |
| 寄存器直读（安全区以外） | `busybox devmem <addr> 32` |
| 精确速率发送 | `cansend2k -i can1 -n 20000 -r 2000 [-R]`（`-R`=SCHED_FIFO 压抖动） |
| 逐帧间隔/抖动 | `canrx can1 60`（输出 avg/min/max 与直方图） |
| 对账丢帧 | 发送端 ID 递增（0x001..），接收端看 ID 连续性与总数 |

> 注意：`candump` 在高帧率下自身会丢，**不要用它做丢帧判据**；用 ID 连续性 + 驱动计数器。

---

## 6. 最终验收数据（纯中断）

```
poll_us=0（纯中断）
cansend2k -n 20000 -r 2000    → 20000 / 0 帧，实际 1999.9 fps（-0.01%），jitter 0.57 ms
/proc/interrupts              → c2f0000.gpio 25 : 96 → 5025（+4929）
RX（loopback 双份）           → +40000 帧，rx_errors +0
空闲 CPU                      → 0.61%（8 核合计，轮询方案是 8% 单核）
真实总线（loopback off）      → 20000/20000，tx_errors 0，bus-errors 0
```

---

## 7. 结论 / 注意事项

1. **中断能通的前提是 BCT 的 `pin-2-int-line`** —— 这是纯 BSP 层问题，内核侧无法弥补（`request_irq` 成功但硬件不产生中断）
2. **验证中断必须关轮询**（`poll_us=0`），否则电平中断被轮询"吃掉"
3. **SPI 上限约 12 MHz**（>=15 MHz 探针失败，失败后需 1 MHz 探针救回）
4. **2 kHz 时 SPI 不是瓶颈**（FD 模式每帧对象 76 B，2 kHz 只占 10 MHz 总线约 14%）
5. 中断不通时的兜底：`poll_us=200 gpio_poll=1`（门控轮询，空闲 ~1%，2 kHz 仍零丢帧）
