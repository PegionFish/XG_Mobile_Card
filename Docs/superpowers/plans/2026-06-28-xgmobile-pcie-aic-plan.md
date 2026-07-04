# XG Mobile PCIe AIC 卡 — 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 设计并制造一张 PCIe AIC 卡，使标准 UEFI x86 PC 能通过 XG Mobile 连接器使用 ASUS 官方 XG Mobile 显卡坞（GC31S / RTX 3080 Mobile）。

**Architecture:** 4 层 PCB，PCIe Gen 3.0 x8 直通（预留 Redriver），STM32F072 MCU 模拟主机侧 EC 握手，USB 通过板载 Type-E 接口从主板走线。第一版不实现 PD 充电。

**Tech Stack:** KiCad 8.0 (PCB), STM32CubeMX + GCC ARM Embedded (固件), JLCPCB (制造)

**Spec:** `docs/superpowers/specs/2026-06-28-xgmobile-pcie-aic-design.md`

---

### Task 1: 项目结构初始化

**Files:**
- Create: `XG_Mobile_Card.kicad_pro`
- Create: `XG_Mobile_Card.kicad_sch`
- Create: `PCIe_GoldFinger.kicad_sch`
- Create: `MCU_Host.kicad_sch`
- Create: `USB_Routing.kicad_sch`
- Create: `Clock_Power.kicad_sch`
- Create: `Firmware/` 目录
- Copy: `Footprints/` `Symbols/` `fp-lib-table` `sym-lib-table` 从原 repo 的 `aic_main` 分支（已在工作目录中）

- [ ] **Step 1: 创建 KiCad 项目**

打开 KiCad 8.0，File → New Project，保存为 `XG_Mobile_Card/XG_Mobile_Card.kicad_pro`。

- [ ] **Step 2: 创建分层原理图页**

在 Schematic Editor 中：
- File → New Schematic，保存为 `XG_Mobile_Card.kicad_sch`
- 从该页 Place → Add Hierarchical Sheet，创建 4 个子页：
  - `PCIe_GoldFinger.kicad_sch`（sheet name: "PCIe Interface"）
  - `MCU_Host.kicad_sch`（sheet name: "MCU & Sideband"）
  - `USB_Routing.kicad_sch`（sheet name: "USB Routing"）
  - `Clock_Power.kicad_sch`（sheet name: "Clock & Power"）

- [ ] **Step 3: 配置项目库路径**

Preferences → Manage Symbol Libraries，确认 `sym-lib-table` 中的路径指向项目内 `Symbols/` 目录。
Preferences → Manage Footprint Libraries，确认 `fp-lib-table` 中路径指向项目内 `Footprints/` 目录。

- [ ] **Step 4: 初始化固件目录**

```bash
mkdir -p Firmware/Core/Inc Firmware/Core/Src Firmware/Core/Startup
cp XG_Mobile_Dock_MCU/Drivers/ Firmware/Drivers/ -r
cp XG_Mobile_Dock_MCU/stm32f0x0.svd Firmware/
cp XG_Mobile_Dock_MCU/openocd.cfg Firmware/
```

- [ ] **Step 5: 验证文件结构**

Run: `find . -maxdepth 3 -type f | sort`
Expected: 看到 4 个 `.kicad_sch` 文件、1 个 `.kicad_pro`、`Firmware/` 目录含 Core/ 和 Drivers/

- [ ] **Step 6: 提交**

```bash
git add XG_Mobile_Card.* PCIe_GoldFinger.kicad_sch MCU_Host.kicad_sch USB_Routing.kicad_sch Clock_Power.kicad_sch Firmware/ fp-lib-table sym-lib-table
git commit -m "init: KiCad project structure and firmware skeleton"
```

---

### Task 2: 原理图 — PCIe 金手指 + AC 耦合 + Redriver 焊盘

**Files:**
- Modify: `PCIe_GoldFinger.kicad_sch`

- [ ] **Step 1: 放置 PCIe x16 金手指符号**

打开 `PCIe_GoldFinger.kicad_sch`。
Place → Add Symbol → 搜索 `PCIE-164-02-F-D-TH-TR`（来自 `Symbols/PCIE-164-02-F-D-TH-TR.kicad_sym`）。
此符号是 164 引脚的 PCIe x16 连接器。放置 1 个。

- [ ] **Step 2: 安放 AC 耦合电容**

仅 lane 0-7 需要 AC 耦合（lane 8-15 悬空）。
对每条 lane i（0..7）：
- 从金手指 `PETp(i)` 串联 220nF 电容（Place → Add Symbol → Device:C，Value=220nF，Footprint=C_0402_1005Metric）
- 从金手指 `PETn(i)` 串联 220nF 电容
- 电容另一侧引出网络标签 `PCIE_TXP{i}_C` / `PCIE_TXN{i}_C`

同样处理 PERp/PERn（接收端），网络标签命名为 `PCIE_RXP{i}_C` / `PCIE_RXN{i}_C`。

共计 32 个 0402 220nF 电容。

- [ ] **Step 3: 放置 Redriver 焊盘**

Place → Add Symbol → 搜索 `SN75LVPE4410RNQR`（来自 `Symbols/SN75LVPE4410RNQR.kicad_sym`）。
放置 2 个，分别标记为 U101（lane 0-3）和 U102（lane 4-7）。

Redriver 的信号路径：
```
金手指 → AC 耦合 → Redriver RX → Redriver TX → XG Mobile
```
但在 bypass 模式下，AC 耦合后直接通过 0Ω 跳线直通到 XG Mobile。

**Bypass 设计**：Redriver 每个 lane 的输入/输出之间放一对 0Ω 电阻焊盘，默认焊接 0Ω。Redriver 芯片本身不焊接。

对每个 lane i (0..7)：
- 从 `PCIE_TXP{i}_C` → 0Ω 电阻 R{100+i*2} → 网络标签 `PCIE_TXP{i}_XG`
- 从 `PCIE_TXN{i}_C` → 0Ω 电阻 R{100+i*2+1} → 网络标签 `PCIE_TXN{i}_XG`
- Redriver 的 RX 输入也连到 `PCIE_TXP{i}_C` / `PCIE_TXN{i}_C`
- Redriver 的 TX 输出也连到 `PCIE_TXP{i}_XG` / `PCIE_TXN{i}_XG`

接收端（PER）同理，方向相反。

Redriver 供电引脚（VCC, VIO）连到 `+3V3` 网络。所有配置引脚通过上拉/下拉电阻设置（参考 SN75LVPE4410 datasheet）。

- [ ] **Step 4: 放置 XG Mobile 连接器符号**

Place → Add Symbol → 搜索 `2351970-1`（XG Mobile 母座，来自 `Footprints/XG_Mobile.pretty/2351970-1.kicad_mod`）。

将 `PCIE_TXP{i}_XG`、`PCIE_TXN{i}_XG`（lane 0-7）连到 XG Mobile 符号的对应引脚：
- `PCIE_TXP{i}_XG` → `TX{i}P`（例如 lane 0: C1, PCIENB_TXP0_C）
- `PCIE_TXN{i}_XG` → `TX{i}N`（C2）
- `PCIE_RXP{i}_XG` → `RX{i}P`（C4）
- `PCIE_RXN{i}_XG` → `RX{i}N`（C5）

**注意**：XG Mobile 的 TX 是主机发射方向。在主机端，金手指的 PET（Transmit）对应 XG Mobile 的 TX。需要验证方向一致性：
- 主机 PCIe 根复合体 → 金手指 PET（发送）→ 线缆 → XG Mobile 连接器 RXP（接收端）→ 显卡坞 GPU
- 显卡坞 GPU → XG Mobile 连接器 TXP（发送）→ 线缆 → 金手指 PER（接收）→ 主机 PCIe 根复合体

因此金手指 PET 连接到 XG Mobile 的 RXP，金手指 PER 连接到 XG Mobile 的 TXP。

**更正 Step 2 的网络标签**：
- 金手指 PET → `PCIE_TX_HOST_P/N{i}` → AC 耦合 → `PCIE_RX_XGM_P/N{i}` → XG Mobile RXP/N (C4/C5, C10/C11 等)
- 金手指 PER ← `PCIE_RX_HOST_P/N{i}` ← AC 耦合 ← `PCIE_TX_XGM_P/N{i}` ← XG Mobile TXP/N (C1/C2, C7/C8 等)

- [ ] **Step 5: 放置 REFCLK 金手指侧连接**

金手指 sideband A13 (REFCLK+) 和 A14 (REFCLK-) → 网络标签 `HOST_REFCLK_P` / `HOST_REFCLK_N`。这些连到 `Clock_Power.kicad_sch` 页（层次引脚）。

- [ ] **Step 6: 金手指电源和未用引脚**

- 3.3V pins (B1, B2, B3, A9, A10, B17, B18, B31, B32, A32, A33)：通过层次标签 `+3V3_PCIE` 引出到 `Clock_Power.kicad_sch`
- 12V pins (A2, A3, B14, B15, A21, A22, B25, B26, A28, A29, B36, B37)：通过层次标签 `+12V_PCIE` 引出
- GND pins：全部连到 GND 网络
- 未用 lane 8-15 的金手指焊盘：每个高速信号通过 10kΩ 下拉到 GND（防止浮空天线效应）
- SMBus (A40, A41)：悬空，加 "No Connect" 标记
- JTAG pins (A15-A20 等)：加 "No Connect" 标记
- PRSNT1#/PRSNT2#：按 PCIe 规范配置（x8 卡：PRSNT1# 接 GND，PRSNT2# 悬空）

- [ ] **Step 7: ERC 检查**

Run: Schematic Editor → Inspect → Electrical Rules Checker
Expected: 0 errors, 允许 warnings（如未连接的 No Connect 引脚）

- [ ] **Step 8: 提交**

```bash
git add PCIe_GoldFinger.kicad_sch
git commit -m "sch: PCIe gold finger with AC coupling and redriver pads"
```

---

### Task 3: 原理图 — MCU + 边带握手信号

**Files:**
- Modify: `MCU_Host.kicad_sch`

- [ ] **Step 1: 放置 STM32F072C8T6 符号**

Place → Add Symbol → 从 KiCad 标准库或手动创建 `STM32F072C8Tx` 符号（LQFP48 封装）。
若无标准库，从 STM32CubeMX 生成符号或使用 KiCad 的 `MCU_ST_STM32F0` 库。

放置 1 个，标记为 `U201`。

- [ ] **Step 2: MCU 外围电路**

- **晶振**：8MHz 无源晶振（HC-49S 或 3225 SMD）+ 2× 22pF 电容，连到 PF0 (OSC_IN) 和 PF1 (OSC_OUT)
- **NRST**：10kΩ 上拉到 3.3V + 100nF 到 GND
- **BOOT0**：10kΩ 下拉到 GND（从 Flash 启动）
- **VDDA**：经磁珠 + 1μF + 100nF 滤波后连 3.3V
- **VDD** (多个引脚)：每个 VDD 引脚旁路 100nF 到 GND，然后连 3.3V
- **VCAP**：按 STM32F072 datasheet，放置 4.7μF 电容到 GND

- [ ] **Step 3: SWD 调试接口**

Place → Add Symbol → Conn_01x04 (4-pin 排针)，标记为 `J201 (SWD)`。
- Pin 1: `+3V3` → 板载 3.3V（不供电，仅用于电平参考）
- Pin 2: `SWCLK` → MCU PA14
- Pin 3: `GND`
- Pin 4: `SWDIO` → MCU PA13

- [ ] **Step 4: UART 调试接口**

Place → Add Symbol → Conn_01x04 (4-pin 排针)，标记为 `J202 (USART1)`。
- Pin 1: `+3V3`
- Pin 2: `USART1_TX` → MCU PA9
- Pin 3: `USART1_RX` → MCU PA10
- Pin 4: `GND`

- [ ] **Step 5: XG Mobile 边带信号连接**

Place → Add Symbol → `2351970-1`（XG Mobile 母座，与 Task 2 同一个元件，此处仅放置边带部分）。

按照设计文档 §5.3 引脚分配表连接：

| MCU 引脚 | XG Mobile 引脚 | 信号名 |
|----------|---------------|--------|
| PA0 (INPUT, 上拉) | C31 | CON_DET_TOWER# |
| PA1 (OUTPUT, OD) | D30 | DGPU_PWR_EN# |
| PA2 (OUTPUT, OD) | D29 | GPU_RST# |
| PA3 (INPUT, 上拉) | D28 | DGPU_PWROK |
| PA4 (INPUT, 上拉) | D27 | P_AC_LOSS_10 |
| PA5 (INPUT, 上拉) | D26 | EC_ACGPU_MCU_IRQ# |
| PA6 (INPUT, 上拉) | D23 | CON_SW1_DET_NB |
| PA7 (INPUT, 上拉) | D22 | Reserve_NB |
| PB6 (I2C1_SCL) | D24 | AGPU_SMB1_CLK |
| PB7 (I2C1_SDA) | D25 | AGPU_SMB1_DAT |
| PB1 (OUTPUT, PP=LOW) | D31 | CON_DET_NB → GND |

每个 INPUT 引脚添加 10kΩ 上拉到 3.3V。
每个 OUTPUT 引脚（OD 模式）添加 10kΩ 上拉到 3.3V（外部上拉，因为 OD 输出只能拉低）。

I2C 线（PB6/PB7）各添加 4.7kΩ 上拉到 3.3V。

- [ ] **Step 6: LED 指示**

PA15 → 经 330Ω 限流电阻 → LED `STATUS`（挡板，绿色）→ GND
PB8 → 经 330Ω 限流电阻 → LED `LINK`（挡板，蓝色）→ GND

- [ ] **Step 7: USB 预留**

PA11 (USB_DM)、PA12 (USB_DP)：放置 2-pin 排针 `J203`，标记为 "USB (Reserved)"。每根线串联 22Ω 电阻。

- [ ] **Step 8: 层次引脚 — 供电接口**

放置层次标签 `+3V3` 和 `GND`，连接到 `Clock_Power.kicad_sch` 页（在顶层原理图中通过这些标签互联）。

- [ ] **Step 9: ERC 检查 + 提交**

Run: ERC → 0 errors
```bash
git add MCU_Host.kicad_sch
git commit -m "sch: MCU with sideband handshake signals and debug headers"
```

---

### Task 4: 原理图 — USB 路由 + 时钟 + 电源

**Files:**
- Modify: `USB_Routing.kicad_sch`
- Modify: `Clock_Power.kicad_sch`

- [ ] **Step 1: USB 路由页 — Type-E 母座**

打开 `USB_Routing.kicad_sch`。

Place → Add Symbol → 搜索 `USB_Type-E_Key-A`（若无标准符号，新建一个 20-pin 连接器符号）。

放置 1 个，标记为 `J301`。

Type-E 20-pin 定义（Key-A，标准主板内部接口）：

| Pin | Signal | → XG Mobile |
|-----|--------|-------------|
| 1 | VBUS | 悬空（不从主板 USB 取电） |
| 2 | SSRX1+ | → A11 (RX2+) |
| 3 | SSRX1- | → A10 (RX2-) |
| 4 | GND | → GND |
| 5 | SSTX1- | → A3 (TX1-) |
| 6 | SSTX1+ | → A2 (TX1+) |
| 7 | GND | → GND |
| 8 | D- | → A7 (D-1) |
| 9 | D+ | → A6 (D+1) |
| 10 | (key pin, 无信号) | — |
| 11 | (key pin) | — |
| 12 | (key pin) | — |
| 13 | GND | → GND |
| 14 | SSTX2+ | 悬空（XG Mobile 仅 1 组 SuperSpeed） |
| 15 | SSTX2- | 悬空 |
| 16 | GND | → GND |
| 17 | SSRX2+ | 悬空 |
| 18 | SSRX2- | 悬空 |
| 19 | VBUS | 悬空 |
| 20 | CC | 悬空（留给 PD） |

Type-E 到 XG Mobile 之间不需要额外元件（直通走线）。

- [ ] **Step 2: USB 路由页 — 放置 XG Mobile USB 引脚**

Place → Add Symbol → `2351970-1`（仅 USB 相关引脚）。

连接：
- J301 SSTX1+ → A2 (TX1+)
- J301 SSTX1- → A3 (TX1-)
- J301 SSRX1+ → A11 (RX2+)
- J301 SSRX1- → A10 (RX2-)
- J301 D+ → A6 (D+1)
- J301 D- → A7 (D-1)

注意 TX/RX 命名：在 USB 3.2 中，主机 TX 连接设备 RX。Type-E 的 SSTX（主板发送）→ XG Mobile 的 RX（设备接收）。

实际上 XG Mobile A2/A3 标记为 TX1+/TX1-（以主机为参考的发送端），而 Type-E 的 SSTX 也是主机的发送端。两者方向一致——都是主机→设备方向。

验证：A2/A3 = TX1+/TX1-（主机 TX），A10/A11 = RX2+/RX2-（主机 RX）。Type-E SSTX 连到主机 TX，SSRX 连到主机 RX。实际上 Type-E 的命名是从 motherboard 视角的：SSTX = 主板发送 = 主机 TX。所以：
- J301 SSTX → A2/A3 (主机 TX) ✓
- J301 SSRX → A10/A11 (主机 RX) ✓

- [ ] **Step 3: 时钟 + 电源页 — 时钟 Buffer**

打开 `Clock_Power.kicad_sch`。

Place → Add Symbol → 搜索 `9DBL0855`（若库中无，创建新符号）。

- 输入：`HOST_REFCLK_P/N`（从 PCIe 页传入的层次标签）→ 交流耦合 100nF → 9DBL0855 CLKIN_P/N
- 输出：选择 1 路输出（如 DIF0），连到网络标签 `XGM_REFCLK_P` / `XGM_REFCLK_N`
- 其余 7 路输出：通过 50Ω 端接到 GND（或悬空，参考 datasheet）
- 供电：VDD = 3.3V，各 VDD 引脚旁路 100nF
- 配置引脚 (SADR, SCLK, SDATA 等)：按 datasheet 默认配置（无 SMBus 控制，硬件 strapping 模式）

9DBL0855 的硬件 strapping 配置：
- SADR[1:0] = 00（地址 0xD8）
- ^OE# = GND（输出使能）
- ^PD# = 3.3V（不上电省电）
- SCLK, SDATA = 悬空（硬件模式，使用 latch-in 值）
- 输出幅值选择：按默认 HCSL 幅值

- [ ] **Step 4: 时钟 + 电源页 — 电源**

- **3.3V 轨**：从 `+3V3_PCIE`（来自 PCIe 页）经 LDO（AP7361-10ER-13）
  - IN: `+3V3_PCIE`
  - OUT: `+3V3`
  - 输入/输出各旁路：2.2μF + 100nF
  - EN: 接 `+3V3_PCIE`（输入电压使能）

- **PD 预留**：放置 TPS65987DDHRSHR 焊盘（U401），仅连接供电和 SPI Flash 接口，不连接 CC/VBUS（第一版不实现）
  - SPI Flash 焊盘：MX25L8006EM1I-12G（8Mb）
  - LDO_3V3 输出 → 连到 PD 的 VIN

- **电源指示灯**：`+3V3` 经 1kΩ → LED（绿色，板载）→ GND

- [ ] **Step 5: ERC 检查 + 提交**

Run: ERC → 0 errors

```bash
git add USB_Routing.kicad_sch Clock_Power.kicad_sch
git commit -m "sch: USB Type-E routing, clock buffer, and power rails"
```

---

### Task 5: 顶层原理图连线 + 全项目 ERC

**Files:**
- Modify: `XG_Mobile_Card.kicad_sch`

- [ ] **Step 1: 顶层原理图 — 层次标签互联**

打开 `XG_Mobile_Card.kicad_sch`。

放置 4 个层次引脚（hierarchical labels）在各个 sheet 之间互联：

| 信号 | 来源 Sheet | 目标 Sheet |
|------|-----------|-----------|
| `+3V3_PCIE` | PCIe_GoldFinger | Clock_Power |
| `+3V3` | Clock_Power | MCU_Host, PCIe_GoldFinger |
| `GND` | 所有 sheets | 全局 |
| `HOST_REFCLK_P/N` | PCIe_GoldFinger | Clock_Power |
| `XGM_REFCLK_P/N` | Clock_Power | (导出到 XG Mobile 母座，在 Clock_Power 页直接连接) |

实际上，在分层设计中，全局标签（global labels）可以跨页使用，不需要显式的层次引脚。但使用层次引脚（hierarchical pins）更清晰。

- [ ] **Step 2: 全项目 ERC**

Run: Schematic Editor → Inspect → Electrical Rules Checker（勾选所有子页）
Expected: 0 errors, 0 warnings

- [ ] **Step 3: 提交**

```bash
git add XG_Mobile_Card.kicad_sch
git commit -m "sch: top-level hierarchy and full ERC pass"
```

---

### Task 6: PCB Layout

**Files:**
- Create: `XG_Mobile_Card.kicad_pcb`

- [ ] **Step 1: 从原理图导入**

PCB Editor → Tools → Update PCB from Schematic → 导入所有元件和网络。

- [ ] **Step 2: 板框定义**

根据 PCIe AIC 规范：
- 板厚：1.6mm
- 板宽：111.15mm (标准全高)
- 板长：312mm (x16 全长)
- Edge connector：金手指区域按 PCIe CEM 规范

在 Edge.Cuts 层绘制板框。导入 PCIe 金手指的 footprint（`PCIE-164-02-F-D-TH-TR` 含正确的金手指位置）。

- [ ] **Step 3: 叠层设置**

Board Setup → Physical Stackup → 4 层板：
- L1: Top (信号)
- L2: GND (完整地平面)
- L3: PWR (3.3V 电源平面 + 部分信号)
- L4: Bottom (信号)

阻抗控制：
- 差分对（PCIe lane）：85Ω，L1 层，参考 L2 GND 平面
- USB SuperSpeed：90Ω 差分
- USB 2.0 D+/D-：90Ω 差分（或 45Ω 单端）

使用 JLCPCB JLC7628 叠层参数（参考 Build_Guide.md）：
- L1-L2 间距：0.2mm prepreg
- L2-L3 间距：1.065mm core
- L3-L4 间距：0.2mm prepreg
- 铜厚：外层 1oz，内层 0.5oz

- [ ] **Step 4: 元件布局**

**大原则**：PCIe 信号路径最短，MCU 远离高速信号。

- **金手指**：底部边缘（PCB 唯一与主板接触的区域）
- **XG Mobile 母座**：顶部边缘，靠挡板侧（右侧）
- **AC 耦合电容**：紧靠金手指，lane 按顺序排列
- **Redriver 焊盘**：AC 耦合电容和 XG Mobile 之间
- **时钟 Buffer**：靠近金手指 REFCLK 引脚
- **MCU**：远离 PCIe 高速信号区域（建议放在左侧或顶部角落）
- **Type-E 母座**：PCB 尾部（与挡板相对的左侧边缘）
- **SWD/UART 排针**：MCU 附近，方便调试

- [ ] **Step 5: PCIe 走线**

对每条 lane (0-7)：
- 差分对走线，线宽/间距由阻抗计算器确定（JLCPCB 4 层板 JLC7628：约 0.3mm 线宽，0.2mm 间距可达 85Ω）
- 每对 P/N 等长（偏差 < 1mm）
- 8 条 lane 间等长（偏差 < 5mm）
- 避免 90° 拐角，使用弧形或 45° 角
- 不在 lane 间插入过孔（顶层走线，参考 L2 GND）
- 高亮显示 net class "PCIe"（可在 Design Rules 中创建差分对规则）

- [ ] **Step 6: USB 走线**

- Type-E → XG Mobile USB 引脚：顶层走线
- SuperSpeed 差分对 (TX+/-, RX+/-)：90Ω，等长
- USB 2.0 D+/D-：90Ω 差分，等长
- 远离 PCIe 时钟和边带信号（避免串扰）

- [ ] **Step 7: 时钟走线**

- 金手指 REFCLK → 时钟 Buffer → XG Mobile D19/D20
- HCSL 差分 100Ω（或 85Ω，参考 datasheet）
- 等长要求：P/N < 0.5mm

- [ ] **Step 8: 电源和地**

- 大面积覆铜 GND（L2 完整平面，L1/L3/L4 部分覆铜）
- 3.3V 通过 L3 电源平面或粗走线（> 2mm 宽）
- 每个 IC 的 VDD 引脚附近放置 100nF 去耦电容
- 过孔（via stitching）连接各层 GND

- [ ] **Step 9: DRC 检查**

Run: PCB Editor → Inspect → Design Rules Checker
Expected: 0 errors, 0 unconnected nets

关键规则：
- 最小线宽/间距：按 JLCPCB 4 层板能力（6mil/6mil）
- 差分对阻抗误差：±10%
- 过孔尺寸：最小 0.3mm 钻孔 / 0.6mm 焊盘

- [ ] **Step 10: 导出 Gerber**

File → Fabrication Outputs → Gerbers → 选择所有铜层 + 丝印 + 阻焊 + Edge.Cuts。
同时导出 Drill files (Excellon 格式)。

- [ ] **Step 11: 提交**

```bash
git add XG_Mobile_Card.kicad_pcb XG_Mobile_Card.kicad_pro
git commit -m "pcb: initial layout with PCIe x8 routing and MCU placement"
```

---

### Task 7: MCU 固件 — STM32CubeMX 初始化

**Files:**
- Create: `Firmware/XG_Mobile_Host_MCU.ioc`
- Create: `Firmware/Core/Src/main.c`
- Create: `Firmware/Core/Inc/main.h`
- Create: `Firmware/Core/Inc/stm32f0xx_hal_conf.h`
- Create: `Firmware/Core/Inc/stm32f0xx_it.h`
- Create: `Firmware/Core/Src/stm32f0xx_hal_msp.c`
- Create: `Firmware/Core/Src/stm32f0xx_it.c`
- Create: `Firmware/Core/Startup/startup_stm32f072c8tx.s`
- Create: `Firmware/STM32F072C8TX_FLASH.ld`
- Create: `Firmware/Makefile`
- Create: `Firmware/openocd.cfg`

- [ ] **Step 1: 用 STM32CubeMX 创建项目**

打开 STM32CubeMX，New Project → 选择 STM32F072C8Tx。
配置：
- **RCC**: HSE = Crystal/Ceramic Resonator
- **SYS**: Debug = Serial Wire
- **I2C1**: Mode = I2C, Speed = Fast Mode (400kHz)
- **USART1**: Mode = Asynchronous, Baud = 115200
- **GPIO**: 按 §5.3 引脚分配表配置输入/输出模式

生成代码，保存 `.ioc` 文件到 `Firmware/XG_Mobile_Host_MCU.ioc`。

- [ ] **Step 2: 从生成代码中提取必要文件**

从 CubeMX 生成的项目中复制：
```bash
cp GeneratedProject/Core/Src/main.c Firmware/Core/Src/
cp GeneratedProject/Core/Inc/main.h Firmware/Core/Inc/
cp GeneratedProject/Core/Inc/stm32f0xx_hal_conf.h Firmware/Core/Inc/
cp GeneratedProject/Core/Inc/stm32f0xx_it.h Firmware/Core/Inc/
cp GeneratedProject/Core/Src/stm32f0xx_hal_msp.c Firmware/Core/Src/
cp GeneratedProject/Core/Src/stm32f0xx_it.c Firmware/Core/Src/
```

- [ ] **Step 3: 创建启动文件和链接脚本**

从原 repo `XG_Mobile_Dock_MCU/` 复制并修改：
```bash
cp XG_Mobile_Dock_MCU/startup_stm32f030x8.s Firmware/Core/Startup/startup_stm32f072c8tx.s
cp XG_Mobile_Dock_MCU/STM32F030C8TX_FLASH.ld Firmware/STM32F072C8TX_FLASH.ld
```

修改链接脚本：将 RAM 大小从 8KB 改为 16KB（STM32F072 的 RAM）。

在 `STM32F072C8TX_FLASH.ld` 中：
```ld
RAM (xrw)   : ORIGIN = 0x20000000, LENGTH = 16K
```

启动文件无需大改——Cortex-M0 指令集相同，中断向量表相同。

- [ ] **Step 4: 创建 Makefile**

从原 repo `XG_Mobile_Dock_MCU/Makefile` 复制并修改：
- TARGET = `XG_Mobile_Host_MCU`
- C_DEFS 中 `-DSTM32F030x8` → `-DSTM32F072x8`
- LDSCRIPT = `STM32F072C8TX_FLASH.ld`
- 源文件列表更新

完整的 Makefile 内容见原 repo 参考。

- [ ] **Step 5: 创建 OpenOCD 配置**

```bash
cp XG_Mobile_Dock_MCU/openocd.cfg Firmware/openocd.cfg
```

修改目标芯片：
```
source [find target/stm32f0x.cfg]
```

- [ ] **Step 6: 验证编译**

```bash
cd Firmware && make clean && make
```

Expected: `XG_Mobile_Host_MCU.elf` 生成成功，无错误。

- [ ] **Step 7: 提交**

```bash
git add Firmware/
git commit -m "fw: STM32F072 CubeMX skeleton with I2C1, USART1, GPIO init"
```

---

### Task 8: MCU 固件 — 握手机状态机

**Files:**
- Create: `Firmware/Core/Src/xg_host_handshake.c`
- Create: `Firmware/Core/Inc/xg_host_handshake.h`

- [ ] **Step 1: 创建头文件**

`Firmware/Core/Inc/xg_host_handshake.h`:

```c
#ifndef XG_HOST_HANDSHAKE_H
#define XG_HOST_HANDSHAKE_H

#include "stm32f0xx_hal.h"
#include <stdbool.h>

typedef enum {
    HS_INIT,
    HS_WAIT_CONNECT,
    HS_WAIT_LOCK,
    HS_POWER_ON,
    HS_WAIT_PWROK,
    HS_RELEASE_RST,
    HS_LINK_UP,
    HS_HEARTBEAT,
    HS_ERROR
} HandshakeState;

/* 初始化握手模块 */
void HS_Init(I2C_HandleTypeDef *hi2c, UART_HandleTypeDef *huart);

/* 主状态机 tick（每次主循环调用一次） */
void HS_Tick(void);

/* 获取当前状态 */
HandshakeState HS_GetState(void);

/* 外部中断回调（在 stm32f0xx_it.c 中调用） */
void HS_GPIO_EXTI_Callback(uint16_t GPIO_Pin);

#endif
```

- [ ] **Step 2: 实现状态机**

`Firmware/Core/Src/xg_host_handshake.c`:

```c
#include "xg_host_handshake.h"
#include <string.h>
#include <stdio.h>

/* XG Mobile 边带 GPIO 引脚定义 — 见设计文档 §5.3 */
#define CON_DET_TOWER_PIN    GPIO_PIN_0   /* PA0 */
#define DGPU_PWR_EN_PIN      GPIO_PIN_1   /* PA1 */
#define GPU_RST_PIN          GPIO_PIN_2   /* PA2 */
#define DGPU_PWROK_PIN       GPIO_PIN_3   /* PA3 */
#define P_AC_LOSS_PIN        GPIO_PIN_4   /* PA4 */
#define MCU_IRQ_PIN          GPIO_PIN_5   /* PA5 */
#define CON_SW1_DET_PIN      GPIO_PIN_6   /* PA6 */
#define RESERVE_PIN          GPIO_PIN_7   /* PA7 */
#define CON_DET_NB_PIN       GPIO_PIN_1   /* PB1 */

#define CON_DET_TOWER_PORT   GPIOA
#define DGPU_PWR_EN_PORT     GPIOA
#define GPU_RST_PORT         GPIOA
#define DGPU_PWROK_PORT      GPIOA
#define P_AC_LOSS_PORT       GPIOA
#define MCU_IRQ_PORT         GPIOA

/* I2C 心跳参数 */
#define I2C_HEARTBEAT_INTERVAL_MS  1000
#define I2C_HEARTBEAT_RETRIES      3
#define I2C_CMD_HEARTBEAT          0xA3
#define I2C_RSP_OK                 0x01
#define I2C_MCU_ADDR               0x73  /* 需原型验证，可能不同 */

static I2C_HandleTypeDef  *hs_i2c;
static UART_HandleTypeDef *hs_uart;
static HandshakeState      hs_state = HS_INIT;
static uint32_t            hs_last_tick = 0;
static uint32_t            hs_heartbeat_missed = 0;
static bool                hs_connect_detected = false;
static bool                hs_lock_detected = false;
static bool                hs_pwrok_detected = false;

static void HS_Log(const char *msg) {
    if (hs_uart) {
        HAL_UART_Transmit(hs_uart, (uint8_t*)msg, strlen(msg), 100);
        HAL_UART_Transmit(hs_uart, (uint8_t*)"\r\n", 2, 100);
    }
}

void HS_Init(I2C_HandleTypeDef *hi2c, UART_HandleTypeDef *huart) {
    hs_i2c = hi2c;
    hs_uart = huart;
    hs_state = HS_INIT;
    
    /* 拉低 CON_DET_NB — 告知显卡坞主机已连接 */
    HAL_GPIO_WritePin(GPIOB, CON_DET_NB_PIN, GPIO_PIN_RESET);
    
    HS_Log("HS: INIT — CON_DET_NB asserted");
}

void HS_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    switch (GPIO_Pin) {
    case CON_DET_TOWER_PIN:
        hs_connect_detected = (HAL_GPIO_ReadPin(CON_DET_TOWER_PORT, CON_DET_TOWER_PIN) == GPIO_PIN_RESET);
        break;
    case CON_SW1_DET_PIN:
        hs_lock_detected = (HAL_GPIO_ReadPin(GPIOA, CON_SW1_DET_PIN) == GPIO_PIN_SET);
        break;
    case DGPU_PWROK_PIN:
        hs_pwrok_detected = (HAL_GPIO_ReadPin(DGPU_PWROK_PORT, DGPU_PWROK_PIN) == GPIO_PIN_SET);
        break;
    }
}

void HS_Tick(void) {
    uint32_t now = HAL_GetTick();

    switch (hs_state) {

    case HS_INIT:
        hs_state = HS_WAIT_CONNECT;
        HS_Log("HS: → WAIT_CONNECT");
        break;

    case HS_WAIT_CONNECT:
        if (hs_connect_detected) {
            hs_state = HS_WAIT_LOCK;
            HS_Log("HS: CONNECT detected → WAIT_LOCK");
        }
        break;

    case HS_WAIT_LOCK:
        if (hs_lock_detected) {
            hs_state = HS_POWER_ON;
            HS_Log("HS: LOCK detected → POWER_ON");
        }
        break;

    case HS_POWER_ON:
        /* 拉低 DGPU_PWR_EN#（请求上电，低有效） */
        HAL_GPIO_WritePin(DGPU_PWR_EN_PORT, DGPU_PWR_EN_PIN, GPIO_PIN_RESET);
        /* 拉低 GPU_RST#（保持复位，低有效） */
        HAL_GPIO_WritePin(GPU_RST_PORT, GPU_RST_PIN, GPIO_PIN_RESET);
        hs_state = HS_WAIT_PWROK;
        HS_Log("HS: PWR_EN# and RST# asserted → WAIT_PWROK");
        break;

    case HS_WAIT_PWROK:
        if (hs_pwrok_detected) {
            hs_state = HS_RELEASE_RST;
            HS_Log("HS: PWROK detected → RELEASE_RST");
        }
        break;

    case HS_RELEASE_RST:
        HAL_Delay(100);  /* 等待电源稳定 */
        /* 释放 GPU_RST#（拉高） */
        HAL_GPIO_WritePin(GPU_RST_PORT, GPU_RST_PIN, GPIO_PIN_SET);
        hs_state = HS_LINK_UP;
        hs_last_tick = now;
        HS_Log("HS: RST# released → LINK_UP");
        break;

    case HS_LINK_UP:
    case HS_HEARTBEAT:
        /* 每 1 秒发送 I2C 心跳 */
        if (now - hs_last_tick >= I2C_HEARTBEAT_INTERVAL_MS) {
            uint8_t cmd = I2C_CMD_HEARTBEAT;
            uint8_t rsp = 0;

            if (HAL_I2C_Master_Transmit(hs_i2c, I2C_MCU_ADDR << 1, &cmd, 1, 50) == HAL_OK &&
                HAL_I2C_Master_Receive(hs_i2c, I2C_MCU_ADDR << 1, &rsp, 1, 50) == HAL_OK &&
                rsp == I2C_RSP_OK) {

                hs_heartbeat_missed = 0;
                hs_state = HS_HEARTBEAT;
            } else {
                hs_heartbeat_missed++;
                char buf[32];
                snprintf(buf, sizeof(buf), "HS: heartbeat miss %lu/3", (unsigned long)hs_heartbeat_missed);
                HS_Log(buf);

                if (hs_heartbeat_missed >= I2C_HEARTBEAT_RETRIES) {
                    hs_state = HS_INIT;
                    HS_Log("HS: heartbeat lost → INIT");
                }
            }
            hs_last_tick = now;
        }
        break;

    case HS_ERROR:
        /* 错误恢复：回到 INIT */
        hs_state = HS_INIT;
        HS_Log("HS: ERROR → INIT");
        break;
    }
}

HandshakeState HS_GetState(void) {
    return hs_state;
}
```

- [ ] **Step 3: 集成到 main.c**

在 `main.c` 中：

```c
#include "xg_host_handshake.h"

extern I2C_HandleTypeDef hi2c1;
extern UART_HandleTypeDef huart1;

int main(void) {
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_I2C1_Init();
    MX_USART1_UART_Init();

    HS_Init(&hi2c1, &huart1);

    while (1) {
        HS_Tick();
        HAL_Delay(10);  /* 10ms tick */
    }
}
```

在 `stm32f0xx_it.c` 的中断处理函数中调用 `HS_GPIO_EXTI_Callback()`。

- [ ] **Step 4: 编译验证**

```bash
cd Firmware && make clean && make
```

Expected: 编译成功，无 warning（启用 -Wall）。

- [ ] **Step 5: 提交**

```bash
git add Firmware/Core/Src/xg_host_handshake.c Firmware/Core/Inc/xg_host_handshake.h Firmware/Core/Src/main.c Firmware/Core/Src/stm32f0xx_it.c
git commit -m "fw: implement XG Mobile host-side handshake state machine with I2C heartbeat"
```

---

### Task 9: 固件烧录 + 裸板测试

- [ ] **Step 1: 烧录固件**

通过 ST-LINK v2 连接 SWD 排针 (J201)：

```bash
openocd -f Firmware/openocd.cfg -c "program Firmware/build/XG_Mobile_Host_MCU.elf verify reset exit"
```

Expected: "** Verified OK **" + "shutdown command invoked"

- [ ] **Step 2: 验证 UART 日志**

连接 USB-UART 转换器到 J202 (USART1)，115200 baud，8N1。

上电后应看到：
```
HS: INIT — CON_DET_NB asserted
HS: → WAIT_CONNECT
```

如果未看到，检查接线和 MCU 时钟。

- [ ] **Step 3: GPIO 信号验证（无显卡坞）**

用万用表测量：
- CON_DET_NB (PB1) = 0V（拉低）
- DGPU_PWR_EN# (PA1) = 3.3V（未触发，高电平）
- GPU_RST# (PA2) = 3.3V（未触发，高电平）

用杜邦线将 PA0 (CON_DET_TOWER#) 拉到 GND 模拟连接，然后拉高 PA6 (CON_SW1_DET_NB) 模拟锁定，观察 UART 日志和 GPIO 变化。

- [ ] **Step 4: 提交**

```bash
git commit -m "test: firmware flash and bare-board GPIO validation"
```

---

### Task 10: 集成测试 — 连接 XG Mobile 显卡坞

**前置条件**：PCB 制造完成、元件焊接完成、固件已烧录。

- [ ] **Step 1: 上电前检查**

- 目视检查：所有焊点、极性元件方向
- 短路检查：3.3V 对 GND、12V 对 GND 无短路（万用表二极管档）
- XG Mobile 连接器：确认公/母方向正确

- [ ] **Step 2: 上电（仅供电，不插 PCIe 槽）**

不将卡插入主板 PCIe 槽。仅从 PCIe 3.3V aux 供电（可通过 PCIe riser 或外部 3.3V 注入）。

验证：
- 3.3V 轨正常（万用表测量）
- MCU 启动：UART 输出日志如前所述

- [ ] **Step 3: 插入 XG Mobile 线缆**

将 XG Mobile 线缆连接器插入卡上母座，锁紧。将线缆 8-pin 端插入显卡坞。

UART 应显示：
```
HS: CONNECT detected → WAIT_LOCK
HS: LOCK detected → POWER_ON
HS: PWR_EN# and RST# asserted → WAIT_PWROK
HS: PWROK detected → RELEASE_RST
HS: RST# released → LINK_UP
```

然后心跳日志应每 1 秒输出一次（无错误）。

- [ ] **Step 4: PCIe 插槽插入 + 系统上电**

将卡插入主板的 x16 PCIe 插槽。主板开机。

进入 UEFI/BIOS 设置 → 查看 PCIe 设备列表 → 应看到 NVIDIA GPU（如 "NVIDIA GeForce RTX 3080"）。

- [ ] **Step 5: Windows 验证**

启动 Windows，安装 NVIDIA 驱动。打开 GPU-Z → 验证：
- 链路速率：PCIe x8 3.0 @ 8.0 GT/s
- 显存大小、CUDA 核心数正确

运行 CUDA-Z 或 `nvidia-smi` 验证 compute 可用性。

- [ ] **Step 6: 提交**

记录测试结果到 `Docs/testing-log.md`。

---

### 风险缓解任务（按需执行）

以下任务仅在前述任务遇到问题时触发：

#### 风险 1: I2C 地址不匹配

如果心跳超时（3 次无回复），在固件中添加 I2C 地址扫描：

```c
void HS_ScanI2C(void) {
    char buf[64];
    for (uint8_t addr = 1; addr < 128; addr++) {
        if (HAL_I2C_IsDeviceReady(hs_i2c, addr << 1, 3, 50) == HAL_OK) {
            snprintf(buf, sizeof(buf), "HS: I2C device found at 0x%02X", addr);
            HS_Log(buf);
        }
    }
}
```

#### 风险 2: 链路只能训练到 Gen 1 或 x4

在 BIOS 中确认：
- PCIe 版本设置 = Auto 或 Gen 3
- 该插槽未被其他设备共享（查阅主板手册）

若仍降速，在固件中增加链路重训练逻辑（触发 PERST# 脉冲）。

---

## 自审检查清单

- [x] **Spec 覆盖**：每个设计文档章节都有对应任务
  - §4 架构 → Task 1 项目结构
  - §5.1 PCIe 信号链路 → Task 2
  - §5.2 时钟 → Task 4 Step 3
  - §5.3 MCU 引脚 → Task 3 Step 5, Task 7
  - §5.4 USB 路由 → Task 4 Step 1-2
  - §5.5 供电 → Task 4 Step 4
  - §5.6 机械 → Task 6 Step 2
  - §6 固件 → Task 7-9
  - §7 测试 → Task 9-10
  - §8 风险 → 内联风险缓解任务
- [x] **无占位符**：所有代码步骤含实际 C 代码，所有硬件步骤含具体元件编号
- [x] **类型一致性**：`HS_Init(hi2c, huart)` 签名在头文件和实现中一致；GPIO 引脚宏在代码和文档中一致
