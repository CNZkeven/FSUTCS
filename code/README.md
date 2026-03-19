# ITEMP 温度控制项目

基于 `STM32F407` 的温度采集与控制工程，包含：

- ADC 温度采样
- PID 控制
- DAC 控制输出
- LCD 本地显示
- 串口参数配置与状态上报

当前工程使用 `CMake` 组织构建，目标固件输出为：

- `build/ITEMP.elf`
- `build/ITEMP.hex`
- `build/ITEMP.bin`

## 目录结构

- `USER/`：主流程、温度换算、控制输出逻辑
- `HARDWARE/`：ADC、DAC、LCD、LED 驱动
- `SYSTEM/`：延时、串口、基础系统封装
- `FWLIB/`：STM32F4 标准外设库
- `tests/`：本地回归测试
- `temperature.md`：温度采样与控制输出链路说明

## 当前硬件口径

### 温度输入

- 温度采样端口固定为 `PA6 / ADC_Channel_6`
- 主循环中每次对 ADC 做 `20` 次平均采样
- 温度显示量程定义为 `0~150℃`
- 外部温度输入经过前端调理后带零点偏置：
  - 外部 `0V` 对应 ADC 约 `2048`
  - 外部 `+5V` 对应 ADC 约 `4095`
- 温度换算公式为：

```text
temperature = max(0, min(adc_value, 4095) - 2048) * 150 / (4095 - 2048)
```

因此当前期望行为是：

- 接地时显示 `0℃`
- 外部 `+5V` 时显示 `150℃`

### DAC 输出

- 控制输出为单极性
- `DAC=2048` 对应外部 `0V`
- `DAC=2048~4095` 线性映射为 `0~5V`
- `DAC=0~2048` 统一视为 `0V`
- 负控制量会被钳位为 `0`，不会输出负电压

控制量到 DAC 码值的当前口径为：

- `0` 控制量 -> `DAC=2048`
- 满量程控制量 -> `DAC=4095`
- 停机/报警时默认输出也回到 `DAC=2048`，即 `0V`

## 最近修正

本轮已经完成两项关键修正：

1. 修正温度换算逻辑  
   旧逻辑误把 `ADC=0` 当成外部 `0V`，导致接地时温度显示偏高。现已改为以 `ADC≈2048` 作为温度零点。

2. 修正 DAC 显示与控制映射  
   旧逻辑按 `0.4V~3.9V` 显示 DAC 输出。现已改为：
   - `2048 -> 0V`
   - `4095 -> 5V`
   - 只使用 DAC 上半段作为有效调压区间

## 本地测试

当前仓库包含两组可直接在本机运行的回归测试：

- `tests/temp_conversion_test.c`
- `tests/control_output_test.c`

运行命令：

```bash
gcc -std=c11 -Wall -Wextra -IUSER tests/temp_conversion_test.c USER/temp_conversion.c -lm -o /tmp/temp_conversion_test && /tmp/temp_conversion_test
gcc -std=c11 -Wall -Wextra -IUSER tests/control_output_test.c USER/control_output.c -lm -o /tmp/control_output_test && /tmp/control_output_test
```

## 固件构建

如果已经完成 CMake 配置，可直接构建：

```bash
env PATH="（你自己的工具路径））/bin:$PATH" cmake --build build
```

说明：

- 这里显式把 STM32 工具链加入 `PATH`
- 否则后处理阶段可能找不到 `arm-none-eabi-objcopy`

## 相关文件

- 温度换算实现：[temp_conversion.c](/code/USER/temp_conversion.c)
- DAC 输出映射实现：[control_output.c](/code/USER/control_output.c)
- 主循环控制逻辑：[main.c](/code/USER/main.c)
- 详细说明文档：[temperature.md](/code/temperature.md)
