# XG Mobile PCIe AIC 卡 — 设计规范

> 日期：2026-06-28 | 状态：草案 | 作者：brainstorming session
> 仓库：https://github.com/PegionFish/XG_Mobile_Card.git

## 1. 目标

设计一张标准 PCIe x16 Add-in Card（AIC），使标准 x86 PC（UEFI）能够通过 XG Mobile 连接器使用 ASUS 官方 XG Mobile 显卡坞。

**当前聚焦场景**：GC31S（RTX 3080 Mobile）显卡坞，用于 CUDA GPGPU 计算任务。

## 2. 约束

- 仅支持 UEFI 固件，不兼容 Legacy BIOS
- PCIe Gen 3.0 x8 优先；预留 Redriver 焊盘 + Gen 4 阻抗以支持后续升级
- 第一版不实现 PD 充电输出，但保留 CC/VBUS 相关 IO 焊盘
- USB 数据通过主板内部 Type-E 接口走线
- Windows 优先，但硬件架构允许后续 Linux 支持

## 3. 技术背景

### 3.1 XG Mobile 连接器信号

从 [Connector.md](../../Docs/Connector.md) 逆向分析得出，XG Mobile 连接器携带：

| 信号类别 | 信号 | 方向（主机视角） |
|---------|------|-----------------|
| PCIe x8 | TXP/N[0:7], RXP/N[0:7] | 双向差分 |
| PCIe 时钟 | GPU_PCIE_CLKP/N (D19/D20) | 主机 → 设备 |
| USB 3.2 Gen 2 | TX+/-, RX+/-, D+/-, SBU1/2, CC1/2 | 双向 |
| 边带 GPIO | CON_DET_TOWER#, DGPU_PWR_EN#, GPU_RST#, DGPU_PWROK, CON_SW1_DET_NB, P_AC_LOSS_10, EC_ACGPU_MCU_IRQ#, Reserve_NB | 混合 |
| SMBus/I2C | AGPU_SMB1_CLK/DAT (D24/D25) | 双向 |
| 连接检测 | CON_DET_NB (D31) | 主机 → 设备（固定 GND） |
| VBUS | 20V USB PD | 设备 → 主机 |

### 3.2 握手协议

从 [MCU.md](../../Docs/MCU.md) 和 [ACPI_Annotated.asl](../../Docs/ACPI_Annotated.asl) 逆向分析得出，主机侧 EC 通过以下序列完成握手：

1. CON_DET_TOWER# 被显卡坞拉低 → EC 检测到物理连接
2. CON_SW1_DET_NB 由显卡坞拉高 → 锁定开关已锁
3. EC 拉低 DGPU_PWR_EN# → 请求 GPU 上电
4. EC 拉低 GPU_RST# → 保持 GPU 复位
5. DGPU_PWROK 由显卡坞拉高 → GPU 电源就绪
6. EC 释放 GPU_RST# → GPU 退出复位
7. I2C 心跳：主机每 1 秒发送 `A3`，MCU 回复 `01`

**关键发现**：不需要 Armoury Crate 软件即可使用显卡坞。握手协议完全由 EC/MCU 在 BIOS/UEFI 枚举 PCIe 前完成。Armoury Crate 仅用于"激活"热插拔——在标准 PC 上，MCU 始终维持握手状态即可，GPU 会被 BIOS 当作标准 PCIe 设备枚举。

### 3.3 现有项目参考

| 参考项 | 文件/位置 | 可复用内容 |
|--------|----------|-----------|
| MCU 硬件设计 | `XG_Mobile_Dock_MCU/` | STM32 外围电路、引脚分配思路 |
| XG Mobile 母座封装 | `Footprints/XG_Mobile.pretty/` | 连接器 Footprint |
| PD 充电器设计 | `USB_PD_Charger.kicad_sch` | TPS65987D 参考电路 |
| PCIe 阻抗控制 | `Docs/Build_Guide.md` | JLCPCB 6 层板参数 |

## 4. 架构

```
┌──── PCIe AIC 卡 (x16 全长, 4 层板, JLC7628 阻抗控制) ────────────────────┐
│                                                                           │
│  主板 x16 插槽                                                            │
│  ├─ PCIe lane 0-7 ── [AC耦合 220nF] ── [Redriver焊盘(空)] ────────────→   │
│  │                                     XG Mobile 母座 (40-pin)             │
│  ├─ REFCLK+/- ── [时钟 Buffer 9DBL0855] ──────────────────────────────→   │
│  │                                     XG Mobile D19/D20                   │
│  ├─ SMBus ── 不使用（MCU 自管 I2C）                                       │
│  ├─ 3.3V ──→ LDO 3.3V ──→ MCU + 时钟 Buffer                             │
│  └─ 12V  ──→ (预留降压)                                                    │
│                                                                           │
│  ┌─ STM32F072C8T6 (LQFP48) ─────────────────────────────────────────┐     │
│  │  GPIO:                                                            │     │
│  │   INPUT:  CON_DET_TOWER#, CON_SW1_DET_NB, P_AC_LOSS_10, MCU_IRQ# │     │
│  │           DGPU_PWROK                                               │     │
│  │   OUTPUT: DGPU_PWR_EN#, GPU_RST#, CON_DET_NB(=GND)                │     │
│  │  I2C:    AGPU_SMB1_CLK/DAT (I2C1, 400kHz Fast Mode)               │     │
│  │  USB:    预留 USB FS PHY → 后续 Host 通信                          │     │
│  │  SWD:    SWCLK/SWDIO → 板载排针 (调试用)                           │     │
│  │  UART:   预留 USART1 TX/RX → 排针 (日志)                           │     │
│  └───────────────────────────────────────────────────────────────────┘     │
│                                                                           │
│  [PCB 尾部]                                                                │
│  └─ USB 3.2 Type-E 母座 (20-pin key-A) ← 排线 ← 主板 Type-E 接口          │
│     USB D+/D-, TX+/-, RX+/- ────────────────────────────────────────→     │
│                                    XG Mobile USB 引脚 (A6/A7, A2/A3 等)    │
│                                                                           │
│  [挡板]                                                                    │
│  ├─ LED × 2: PCIe 链路状态指示 (GPIO 直驱)                                 │
│  └─ USB-C PD 母座 (焊盘预留, 不焊接) ← 后续版本实现 PD 充电输出            │
│                                                                           │
│  [预留焊盘 (不焊接)]                                                        │
│  ├─ Redriver: SN75LVPE4410 × 2 (8 lane)                                  │
│  ├─ PD 控制器: TPS65987D 或 CYPD3177                                      │
│  └─ 相关无源器件                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

## 5. 模块设计

### 5.1 PCIe 信号链路

**拓扑**：金手指 → AC 耦合电容 (220nF, 0402) → 直通走线 → Redriver 焊盘 (bypass) → XG Mobile 母座

**关键参数**：
- 差分阻抗：85Ω ± 10%（目标 Gen 3 8 GT/s）
- 走线等长：同一 lane 的 P/N 对内偏差 < 5mil，lane 间偏差 < 50mil
- AC 耦合电容靠近金手指侧放置

**Redriver 预留**：使用现有项目已验证的 SN75LVPE4410RNQR。8 lane 需 2 颗（每颗支持 4 lane）。焊盘按 TI 推荐布局预留。Bypass 模式下用 0Ω 跳线电阻直通。

### 5.2 时钟方案

- 芯片：**9DBL0855**（8 输出 PCIe 时钟 buffer，HCSL）
- 输入：金手指 REFCLK (sideband A13/A14)
- 输出：1 路 → XG Mobile D19/D20（其余输出悬空或端接）
- 供电：3.3V（来自 PCIe 插槽）
- 备选料：PI6C20800（更低成本，pin 兼容性需验证）

**不使用 SRIS**。Gen 3 公共时钟模式完全够用。

### 5.3 MCU 选型与引脚分配

#### 选型：STM32F072C8T6

| 参数 | 值 |
|------|-----|
| 核心 | Cortex-M0 @ 48MHz |
| Flash | 64KB |
| RAM | 16KB |
| 封装 | LQFP48 (7×7mm) |
| I2C | 2 路（使用 1 路） |
| USB | 内置 FS PHY（预留） |
| 供电 | 3.3V |
| 编程 | SWD (PA13/PA14) |

> STM32F030C8T6 可作为降级备选（无 USB PHY），pin 兼容 LQFP48。

#### 引脚分配

| 引脚 | 功能 | 方向 | 连接目标 |
|------|------|------|---------|
| PA0  | CON_DET_TOWER# | INPUT (EXTI, 双沿中断) | XG Mobile C31 |
| PA1  | DGPU_PWR_EN# | OUTPUT (OD) | XG Mobile D30 |
| PA2  | GPU_RST# | OUTPUT (OD) | XG Mobile D29 |
| PA3  | DGPU_PWROK | INPUT (EXTI) | XG Mobile D28 |
| PA4  | P_AC_LOSS_10 | INPUT (上拉) | XG Mobile D27 |
| PA5  | MCU_IRQ# | INPUT (EXTI, 上拉) | XG Mobile D26 |
| PA6  | CON_SW1_DET_NB | INPUT (上拉) | XG Mobile D23 |
| PA7  | Reserve_NB | INPUT | XG Mobile D22 |
| PA9  | USART1_TX | OUTPUT | 排针 (调试) |
| PA10 | USART1_RX | INPUT | 排针 (调试) |
| PA11 | USB_DM | USB | 预留 |
| PA12 | USB_DP | USB | 预留 |
| PA13 | SWDIO | SWD | 排针 |
| PA14 | SWCLK | SWD | 排针 |
| PA15 | LED_STATUS | OUTPUT | 挡板 LED |
| PB6  | I2C1_SCL | I2C | XG Mobile D24 |
| PB7  | I2C1_SDA | I2C | XG Mobile D25 |
| PB8  | LED_LINK | OUTPUT | 挡板 LED |
| PB1  | CON_DET_NB | OUTPUT (固定 GND) | XG Mobile D31 |

> 注：CON_DET_NB 固定拉低，告诉显卡坞"主机已连接"。不需要 GPIO 控制，可以直接接 GND；但用 GPIO 输出低电平可在调试时断开。

### 5.4 USB 数据路由

```
主板 Type-E 接口 (20-pin key-A)
  │
  │  排线 (USB 3.2 Gen 2, 最长 ~600mm)
  ▼
PCB 尾部 Type-E 母座
  │
  │  板内走线 (90Ω 差分 USB 3.2, 45Ω 单端 USB 2.0)
  ▼
XG Mobile 母座 USB 引脚
  ├─ A6  (D+)  ← D+
  ├─ A7  (D-)  ← D-
  ├─ A2  (TX1+) ← SSTX+
  ├─ A3  (TX1-) ← SSTX-
  ├─ A10 (RX2-) ← SSRX-
  ├─ A11 (RX2+) ← SSRX+
  ├─ A5  (CC1)  ← (预留 PD)
  ├─ B5  (CC2)  ← (预留 PD)
  ├─ A8  (SBU1) ← (预留 PD)
  └─ B8  (SBU2) ← (预留 PD)
```

注意：XG Mobile 连接器内 USB 信号仅 1 组 SuperSpeed 差分对（TX+/-, RX+/-），因此仅支持 **USB 3.2 Gen 2 x1 (10 Gbps)**，不支持 Gen 2x2 (20 Gbps)。

Type-E 接口的 CC/SBU 信号第一版悬空（留给后续 PD 实现）。

### 5.5 供电

| 电压轨 | 来源 | 用途 | 最大电流 |
|--------|------|------|---------|
| 3.3V | PCIe 金手指 3.3V pins | MCU, 时钟 Buffer, LED | ~150mA |
| 12V  | PCIe 金手指 12V pins | 预留 Redriver、PD 控制器 | ~500mA |

MCU 和时钟 Buffer 功耗极低。LDO 选用 AP7361-10ER-13（现有项目使用）或类似型号。PCIe 3.3V 轨额定 3A，远大于需求。

PD 功能如需供电，从 PCIe 12V 经降压模块取电（后续实现）。

### 5.6 机械设计

- **PCB 外形**：标准 PCIe x16 全长 (312mm × 111mm 典型)，4 层板
- **挡板**：标准全高 PCIe 挡板 (120mm × 18.4mm)
- **挡板开孔**：2× LED (3mm) + 1× USB-C (预留，不装)
- **母座位置**：XG Mobile 连接器（I-PEX CABLINE-VS 或 2351970-1）位于 PCB 顶部边缘，垂直于挡板方向
- **Type-E 母座**：PCB 尾部边缘（与挡板相对的一侧），与排线对齐

## 6. 固件设计概要

### 6.1 角色转换

现有 `XG_Mobile_Dock_MCU` 固件扮演 **MCU（设备侧）**，被动响应主机命令。
本卡固件需扮演 **EC（主机侧）**，主动发起握手。

| 现有固件行为 | 本卡固件行为 |
|-------------|-------------|
| 等待 CON_DET_NB 拉低 | 拉低 CON_DET_NB (告知设备连接) |
| 监听 CON_DET_TOWER# | 监听 CON_DET_TOWER# |
| 响应 DGPU_PWR_EN# 变化 | 主动拉低 DGPU_PWR_EN# |
| 等待 DGPU_PWROK → 输出 | 等待 DGPU_PWROK ← 输入 |
| 响应 I2C 命令 | 主动发起 I2C 心跳 (A3 → 01) |

### 6.2 主状态机

```
INIT → 等待 CON_DET_TOWER# 低电平
     → 检查 CON_SW1_DET_NB 高电平 (锁定状态)
     → 拉低 DGPU_PWR_EN# (请求上电)
     → 拉低 GPU_RST# (保持复位)
     → 等待 DGPU_PWROK 高电平
     → 延迟 100ms
     → 释放 GPU_RST# (拉高)
     → I2C 心跳循环 (每秒发送 A3，期望 01 回复)
     → 如心跳丢失超过 3 秒 → 回到 INIT
```

### 6.3 I2C 心跳

- **从设备地址**：`0x73`（参考现有项目 `OwnAddress=115`；但此为开源 MCU 固件的自定义值，官方 XG Mobile 显卡坞 MCU 的实际 I2C 地址**需在原型阶段用逻辑分析仪确认**）
- 命令：`A3` → 期望 MCU 回复 `01`
- 间隔：1000ms
- 超时恢复：连续 3 次无回复 → 重新握手
- 失败回退：若心跳不通，尝试 I2C 地址扫描 (1-127) 探测官方 MCU 实际地址

## 7. 测试策略

### 7.1 单元测试（固件）

- GPIO 输入模拟：用跳线模拟 CON_DET_TOWER#、CON_SW1_DET_NB 的电平变化
- I2C 回环测试：两块板互连，一块扮演 MCU，一块扮演 EC
- UART 日志输出：状态机每一步打印日志

### 7.2 集成测试

- 上电时序：示波器测量 DGPU_PWR_EN# → DGPU_PWROK → GPU_RST# 时序
- PCIe 链路训练：插入显卡坞后，检查 BIOS 是否枚举到 GPU
- GPU-Z / CUDA-Z 验证链路速率和带宽

## 8. 风险与缓解

| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| XG Mobile 线缆信号衰减过大 | 低 | 链路训不上或降速 | Gen 3 直通保守设计；预留 Redriver |
| I2C 心跳时序不匹配 | 中 | 显卡坞不响应 | 逻辑分析仪抓取官方心跳时序对比 |
| MCU 握手与 BIOS 枚举时序冲突 | 中 | GPU 不被识别 | 固件在系统上电后立即完成握手，早于 PCIe 枚举 |
| JLCPCB 4 层板阻抗精度不够 | 低 | 信号质量差 | 参考 Build_Guide 已验证的叠层参数 |
| 不同主板 REFCLK 驱动能力差异 | 低 | 时钟 Buffer 输出不稳定 | 9DBL0855 兼容多数 HCSL 输入规格 |

## 9. 第二阶段功能（暂不实现）

- PD 充电输出（20V 5A USB-C）
- Redriver 焊接 + Gen 4 验证
- MCU USB HID 接口 + Windows 控制面板
- 功率监控日志（MCU ADC 采样）
- Linux 驱动支持

## 10. 参考

- [XG Mobile 连接器信号分析](../../Docs/Connector.md)
- [MCU 握手协议逆向](../../Docs/MCU.md)
- [ACPI 逆向注释](../../Docs/ACPI_Annotated.asl)
- [软件侧逆向](../../Docs/Software.md)
- [电源设计参考](../../Docs/Power.md)
- [PCB 制造指南](../../Docs/Build_Guide.md)
- [现有 MCU 固件](../../XG_Mobile_Dock_MCU/)
- [现有 XGMDriver](../../XGMDriver/)
