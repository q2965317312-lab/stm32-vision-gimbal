# STM32F407VET6 二轴视觉云台

这是一个基于 STM32F407VET6、TMC2209、步进电机和 MaixCAM2 的二轴直驱视觉跟随云台工程。当前版本已经跑通 OLED 显示、双 TMC2209 UART 初始化、Yaw/Pitch 步进驱动、MaixCAM2 串口视觉数据接收、连续跟随控制、数据超时保护和软件限位。

## 当前状态

- 主控：STM32F407VET6 核心板
- 电机驱动：TMC2209，两路共用 USART1，地址分别为 0 和 1
- 电机：两相四线步进电机
- 视觉：MaixCAM2 识别红色目标，通过 UART5 发送目标偏差
- 显示：OLED 显示 X/Y 偏差、视觉有效标志、循环计数、Yaw/Pitch 软件位置
- 控制方式：开环步进直驱，基于视觉偏差输出连续 STEP 脉冲
- 安全保护：MaixCAM 数据超时停机、丢目标停机、Yaw/Pitch 软件限位

## 硬件接线

### STM32 与 TMC2209

| 功能 | STM32 引脚 | 说明 |
| --- | --- | --- |
| TMC UART TX | PA9 / USART1_TX | 通过 1k 电阻接 PDN_UART 节点 |
| TMC UART RX | PA10 / USART1_RX | 接同一个 PDN_UART 节点 |
| Yaw STEP | PA6 | 输出 STEP 脉冲 |
| Yaw DIR | PC4 | 方向控制 |
| Yaw EN | PA7 | 低电平使能 |
| Pitch STEP | PE9 | 输出 STEP 脉冲 |
| Pitch DIR | PE10 | 方向控制 |
| Pitch EN | PE8 | 低电平使能 |
| VMOT | 12V | 电机电源 |
| VIO | 3.3V | 逻辑电源 |
| GND | GND | 必须和 STM32 共地 |

TMC2209 地址：

| MS1 | MS2 | UART 地址 |
| --- | --- | --- |
| 0 | 0 | 0 |
| 1 | 0 | 1 |
| 0 | 1 | 2 |
| 1 | 1 | 3 |

当前固件使用地址 0 和 1。

### STM32 与 MaixCAM2

MaixCAM2 侧 2x6 接口常用引脚：

| MaixCAM2 | STM32 |
| --- | --- |
| U2T | PD2 / UART5_RX |
| U2R | PC12 / UART5_TX，可选 |
| GND | GND |

如果 MaixCAM2 自己供电，不需要从 STM32 给它供电，但必须共地。

### OLED

| 功能 | STM32 引脚 |
| --- | --- |
| SDA | PC0 |
| SCL | PE12 |
| VCC | 3.3V |
| GND | GND |

## 串口协议

MaixCAM2 向 STM32 发送一帧 ASCII 数据：

```text
X100Y-50Z1E
```

含义：

- `X`：目标水平偏差，正数表示目标在画面右侧
- `Y`：目标垂直偏差，正数表示目标在画面上方
- `Z`：是否识别到目标，`1` 为识别到，`0` 为未识别
- `E`：一帧数据结束

STM32 只有在 `Z == 1` 且最近 `250ms` 内收到新数据时才会驱动电机。

## 方向约定

当前实测方向：

- Yaw `DIR=0`：从上往下看顺时针
- Yaw `DIR=1`：从上往下看逆时针
- Pitch `DIR=0`：向上
- Pitch `DIR=1`：向下

固件中定义：

- `X > 0` 时，Yaw 使用 `DIR=0`
- `Y > 0` 时，Pitch 使用 `DIR=0`

## 调参入口

主要参数集中在：

```text
Core/Inc/gimbal_config.h
```

关键参数：

```c
#define VISION_YAW_START_DEADZONE_PX 35
#define VISION_YAW_STOP_DEADZONE_PX 28
#define VISION_PITCH_START_DEADZONE_PX 18
#define VISION_PITCH_STOP_DEADZONE_PX 10

#define VISION_YAW_HALF_PERIOD_MIN_US 150
#define VISION_YAW_HALF_PERIOD_MAX_US 420
#define VISION_YAW_PERIOD_SCALE_US 12
#define VISION_YAW_DAMPING_GAIN 18
#define VISION_YAW_USE_TIM_PWM 1

#define VISION_PITCH_HALF_PERIOD_MIN_US 100
#define VISION_PITCH_HALF_PERIOD_MAX_US 700
#define VISION_PITCH_PERIOD_SCALE_US 12

#define VISION_FILTER_SHIFT 3
#define VISION_UART_TIMEOUT_MS 250

#define YAW_SOFT_LIMIT_STEPS 12000
#define PITCH_SOFT_LIMIT_STEPS 6000
```

调参方向：

- 过冲：增大对应轴的 `HALF_PERIOD_MIN_US`，或减小 `PERIOD_SCALE_US`
- 太慢：减小对应轴的 `HALF_PERIOD_MIN_US` 和 `HALF_PERIOD_MAX_US`
- 中心附近抖动：增大对应轴的 `START_DEADZONE_PX` / `STOP_DEADZONE_PX`，或增大 `VISION_FILTER_SHIFT`
- 反应太迟钝：减小 `VISION_FILTER_SHIFT`
- 可动范围不够：增大对应轴 `SOFT_LIMIT_STEPS`

当前 Yaw STEP 使用 TIM3_CH1 硬件 PWM 输出，PA6 不再由主循环手动翻转，因此 Yaw 的 STEP 频率不会被 OLED 刷新、串口解析和按键扫描明显拖慢。

## 按键在线调参

当前固件支持运行时微调视觉控制参数：

| 按键 | 功能 |
| --- | --- |
| KEY1 | 切换当前调参项 |
| KEY2 | 当前参数减小 |
| KEY3 | 当前参数增大 |
| KEY4 | 当前参数恢复默认值 |

OLED 显示：

| 显示 | 含义 |
| --- | --- |
| `YD` | Yaw 阻尼值 |
| `YM` | Yaw 最小 STEP 半周期，数值越小速度越快 |
| `PD` | Pitch 阻尼值 |
| `PM` | Pitch 最小 STEP 半周期，数值越小速度越快 |

## 软件限位

当前版本是开环步进，没有编码器。固件以上电时的位置作为软件零点，并记录发出的 STEP 数：

- Yaw 范围：`-12000 ~ +12000`
- Pitch 范围：`-6000 ~ +6000`

测试前建议先把云台手动摆到中位，再上电。

## TMC2209 微步配置

当前 TMC2209 `CHOPCONF` 配置为：

```c
0x12000053
```

含义：

- `MRES=2`：64 微步
- `intpol=1`：开启插值

如果需要更快但能接受更明显的步进感，可改成 `0x13000053`，即 32 微步；`0x14000053`，即 16 微步；或 `0x15000053`，即 8 微步。如果需要更丝滑但速度可以慢一些，可改成 `0x10000053`，即 256 微步。

当前电流配置按地址区分：

- 地址 0：Yaw，`IHOLD_IRUN = 0x00050502`
- 地址 1：Pitch，`IHOLD_IRUN = 0x00080402`

## 构建与烧录

推荐使用 VS Code 任务：

- `Build Keil`：使用 Keil ARMCLANG 编译
- `Flash`：通过 ST-LINK 烧录
- `Check Env`：检查构建和烧录环境

也可以在 PowerShell 中执行：

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\build_keil.ps1
```

编译输出：

```text
build/Debug/F407VET6_Gimbal.hex
```

## 测试流程

1. 上电前把云台摆到机械中位。
2. 烧录固件。
3. 确认 OLED 显示 `VISION STEP`。
4. 确认 TMC 初始化显示 `0OK` 和 `1OK`。
5. MaixCAM2 运行红色目标识别脚本。
6. OLED 上 `V` 显示 `1` 时，云台应跟随目标。
7. 拿走目标或断开 MaixCAM TX，云台应在约 `250ms` 内停止。
8. 观察 OLED 上 `Ya` 和 `Pi`，用于调整软件限位范围。

## 后续优化方向

- 增加按键菜单，用按键在线调整死区、速度和限位
- 增加串口调参协议，免反复编译烧录
- 加入回中功能
- 机械上优化 Pitch 重心
- 如果需要更丝滑的视觉效果，可考虑减速结构、闭环步进或无刷云台方案
