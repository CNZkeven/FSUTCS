# 上位机说明

## 1. 功能概述

上位机是燃油供应单元温度控制系统的人机交互端，基于 Python + PySide6 构建，通过串口与 STM32F407 下位机通信，承担以下职责：

- **串口连接管理** — 选择端口和波特率，建立/断开与下位机的通信链路
- **参数下发** — 将设定温度、PID 参数和超温阈值发送到下位机
- **实时数据展示** — 接收 MCU 遥测帧，刷新 KPI 卡片和运行/报警状态灯
- **曲线绘制** — 以相对时间为横轴，实时绘制温度、设定值和输出电压三条曲线
- **数据记录** — 将完整遥测数据附带时间戳写入 `record.csv`，供实验报告分析

## 2. 技术栈

| 技术 | 用途 |
| --- | --- |
| PySide6 (Qt 6) | GUI 框架，提供窗口、控件和事件机制 |
| QtSerialPort | 串口连接、异步读写和信号回调 |
| pyqtgraph | 高性能实时曲线渲染 |
| dataclasses | 遥测数据模型 |
| csv / pathlib | 实验记录导出 |
| json | PID 参数本地持久化 |

## 3. 目录结构与核心文件

```
app/
├── main.py                   # 程序入口
├── settings.json             # PID 参数持久化文件
├── core/                     # 通信与协议层
│   ├── protocol.py           # 协议编解码
│   ├── serial_manager.py     # 串口管理器
│   └── state.py              # 遥测数据模型
├── ui/                       # 界面层
│   ├── main_window.py        # 主窗口（核心业务逻辑）
│   └── widgets.py            # 自定义控件
└── services/                 # 服务层
    ├── logger.py             # CSV 数据记录
    └── settings.py           # JSON 配置读写
```

### 3.1 `main.py` — 程序入口

创建 `QApplication` 和 `MainWindow`，启动 Qt 事件循环。整个文件仅 16 行，不包含业务逻辑。

### 3.2 `core/protocol.py` — 协议编解码

负责上位机与下位机之间的文本协议格式处理，不涉及串口打开和 UI 刷新。

| 函数 | 作用 |
| --- | --- |
| `is_telemetry_line(line)` | 判断一行文本是否为 10 字段遥测帧（检查逗号数是否为 9） |
| `parse_line(line)` | 将 CSV 文本拆分为 `Telemetry` 数据对象 |
| `build_command(cmd, value)` | 构造控制命令字符串，自动补 `\r\n` 结束符 |

**遥测帧格式**（MCU 每 100ms 上报一行）：

```
TEMP,SET,P,I,D,DAC,VOLT,RUN,ALARM,LIMIT
```

**控制命令格式**（上位机下发）：

```
START\r\n          # 启动加热
STOP\r\n           # 停止控制
TEMP:60.00\r\n     # 设置目标温度
P:12.00\r\n        # 设置比例参数
I:0.02\r\n         # 设置积分参数
D:18.00\r\n        # 设置微分参数
LIMIT:75.00\r\n    # 设置超温阈值
GET\r\n            # 请求当前参数
```

### 3.3 `core/serial_manager.py` — 串口管理器

基于 `QSerialPort` 封装，对 UI 层屏蔽底层串口细节。

**核心机制：**

- **接收分帧**：`readyRead` 信号触发时将字节追加到 `_buffer`，以 `\n` 为界切出完整行，通过 `line_received` 信号通知上层。
- **发送队列**：`send_batch()` 接收多条命令列表后，使用 `QTimer` 按 120ms 间隔逐条发送，避免 MCU 端因命令堆积丢帧。
- **信号接口**：`line_received(str)` 和 `status_changed(str)` 两个 Qt 信号，供 `MainWindow` 连接槽函数。

**设置 120ms 发送间隔的原因：** MCU 主循环以约 100ms 为周期，采用逐帧解析命令，连续无间隔发送会导致多条命令在同一个主循环内堆积。

### 3.4 `core/state.py` — 遥测数据模型

定义 `Telemetry` dataclass，包含 10 个字段，与 MCU 上报的 CSV 字段一一对应：

```python
@dataclass
class Telemetry:
    temp: float       # 当前温度
    set_temp: float   # 设定温度
    p: float          # 比例参数
    i: float          # 积分参数
    d: float          # 微分参数
    dac: float        # DAC 码值
    volt: float       # 输出电压
    run: int          # 运行状态 (1=运行)
    alarm: int        # 报警状态 (1=报警)
    limit: float      # 超温阈值
```

### 3.5 `ui/main_window.py` — 主窗口（核心业务逻辑）

整个上位机最核心的文件，承担四类职责：

**1) 串口连接管理**

`_on_connect()` 从 UI 获取端口名和波特率，调用 `SerialManager.connect_port()` 建立连接；`_on_disconnect()` 断开。

**2) 参数下发**

`_apply_params()` 将界面上的设定温度、PID 参数和超温阈值分别构造为命令，通过 `send_batch()` 分时下发到 MCU。下发前会先调用 `_save_settings()` 将参数持久化到 `settings.json`，下次启动时自动恢复。

**3) 实时数据处理**

`_on_line()` 是数据接收的核心入口，每收到一行串口文本会执行：

```
line_received 信号 → _on_line()
  ├─ 非遥测帧 → 追加到日志文本框
  └─ 遥测帧 → parse_line() 解析
       ├─ 更新 KPI 卡片（温度、设定值、电压、DAC）
       ├─ 更新状态灯（运行、报警）
       ├─ 写入 Logger（如正在记录）
       └─ 追加曲线数据点并刷新图表
```

**4) 曲线绘制**

使用 pyqtgraph 绘制三条实时曲线：

- 绿色：当前温度
- 蓝色：设定温度
- 黄色：输出电压

横轴为相对时间（秒），首帧到达时记录 `_t0` 作为零点。每次点击"启动"会清空历史数据，开始新一轮实验曲线。

**界面布局：**

```
┌──────────────┬───────────────────────────────────┐
│  串口连接    │  [温度] [设定] [电压] [DAC]  KPI  │
│  运行控制    │─────────────────────────────────── │
│  参数设置    │                                    │
│  (设定温度)  │       实时曲线 (pyqtgraph)         │
│  (P / I / D) │                                    │
│  (超温阈值)  │───────────────────────────────────│
│  [下发按钮]  │  [开始记录] [停止记录]             │
│  状态灯      │  日志文本框                        │
└──────────────┴───────────────────────────────────┘
     左侧控制区              右侧监控区
```

### 3.6 `ui/widgets.py` — 自定义控件

| 控件 | 作用 |
| --- | --- |
| `PortComboBox` | 串口下拉框，展开时自动刷新可用端口列表 |
| `KpiCard` | 关键指标卡片，显示标题 + 数值 + 单位 |
| `StatusLight` | 状态指示灯，支持 `on`/`off`/`alarm` 三种状态，通过 Qt 动态属性切换样式 |

### 3.7 `services/logger.py` — CSV 数据记录

`Logger` 类管理实验数据的录制生命周期：

- `start(path)` — 创建 CSV 文件，写入表头，开始录制
- `add(data)` — 将 `Telemetry` 附上时间戳写入一行，每行立即 `flush`
- `stop()` — 关闭文件，结束录制
- `export_csv(path)` — 将内存中的记录导出到指定路径

输出的 `record.csv` 格式：

```csv
timestamp,temp,set_temp,p,i,d,dac,volt,run,alarm,limit
2026-03-19 14:30:01,45.21,60.00,12.00,0.02,18.00,2140,1.233,1,0,75.00
```

### 3.8 `services/settings.py` — JSON 配置读写

提供 `load_settings(path)` 和 `save_settings(path, data)` 两个函数，将 PID 参数持久化到 `settings.json`：

```json
{
  "set_temp": 65.0,
  "p": 12.0,
  "i": 0.02,
  "d": 18.0,
  "limit": 75.0
}
```

上位机启动时自动加载上次的参数，避免每次手动重新填写。

## 4. 数据流

整个上位机的数据流可以归纳为两条链路：

**接收链路**（MCU → 上位机）：

```
QSerialPort.readyRead
  → SerialManager._on_ready_read()   缓冲并按行分帧
  → line_received 信号
  → MainWindow._on_line()            判断是否遥测帧
  → protocol.parse_line()            CSV 解析为 Telemetry
  → 更新 KPI / 状态灯 / 曲线 / Logger
```

**发送链路**（上位机 → MCU）：

```
用户点击按钮
  → protocol.build_command()          构造命令字符串
  → SerialManager.send()             单条发送
     或 SerialManager.send_batch()    批量分时发送（120ms 间隔）
  → QSerialPort.write()              字节写入串口
```

## 5. 运行方式

```bash
# 安装依赖
pip install -r requirements.txt

# 启动上位机
python app/main.py
```

串口参数：115200 波特率、8 数据位、1 停止位、无校验、无流控。

## 6. 核心代码速查表

| 想了解的内容 | 文件 | 关键函数/类 |
| --- | --- | --- |
| 程序入口 | `main.py` | `main()` |
| 串口协议格式 | `core/protocol.py` | `parse_line()`, `build_command()`, `is_telemetry_line()` |
| 串口连接与收发 | `core/serial_manager.py` | `SerialManager.connect_port()`, `_on_ready_read()`, `send_batch()` |
| 遥测数据结构 | `core/state.py` | `Telemetry` dataclass |
| 主界面与业务逻辑 | `ui/main_window.py` | `MainWindow._on_line()`, `_apply_params()`, `_on_connect()` |
| 自定义控件 | `ui/widgets.py` | `KpiCard`, `StatusLight`, `PortComboBox` |
| 数据记录 | `services/logger.py` | `Logger.start()`, `Logger.add()`, `Logger.stop()` |
| 参数持久化 | `services/settings.py` | `load_settings()`, `save_settings()` |