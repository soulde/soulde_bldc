# 系统整体介绍

本仓库是 VESC 电机控制器固件，面向 STM32F4 + ChibiOS 实时系统，主要使用 C 语言实现。系统核心目标是完成 BLDC/FOC 电机控制、通信配置、传感器采集、应用输入处理、故障保护和固件扩展。

系统启动入口在 `main.c`。启动后依次完成 RTOS/HAL 初始化、硬件 GPIO 初始化、配置读取、Flash 完整性检查、电机控制初始化、通信初始化、应用配置加载、CAN/NRF/IMU/超时保护等模块启动，最后进入 ChibiOS 线程调度。

```text
STM32F4 / ChibiOS HAL
        |
hwconf + driver
        |
motor + encoder + imu
        |
comm + conf_general
        |
applications + LispBM/QML
```

## 启动流程

1. `halInit()`、`chSysInit()` 初始化 ChibiOS 和底层 HAL。
2. `HW_EARLY_INIT()`、`hw_init_gpio()` 初始化板级硬件。
3. `mempools_init()`、`events_init()`、`timer_init()` 初始化基础服务。
4. `conf_general_init()` 初始化配置系统。
5. `flash_helper_verify_flash_memory()` 校验固件 Flash 完整性。
6. `mc_interface_init()` 初始化电机控制接口。
7. `commands_init()` 初始化命令协议处理。
8. 根据配置启动 USB、UART、CAN、NRF 等通信。
9. 创建 LED、周期状态上报、Flash 后台检查线程。
10. 启动 `timeout`、`shutdown`、`imu` 等保护和辅助模块。

# 系统组成模块

## 1. 主控与系统调度模块

相关文件包括 `main.c`、`main.h`、`events.c`、`timeout.c`。

该模块负责系统启动、模块初始化、后台线程创建、初始化状态维护和异常复位。`main.c` 中的 `led_thread` 根据运行状态和故障码控制 LED，`periodic_thread` 周期发送转子位置、PID 位置和 FOC 调试数据，`flash_integrity_check_thread` 后台分块检查 Flash 完整性。

## 2. 电机控制模块

相关目录为 `motor/`，核心文件包括 `mc_interface.c`、`mcpwm.c`、`mcpwm_foc.c`、`foc_math.c`。

该模块提供统一电机控制接口 `mc_interface_*`，支持占空比、电流、刹车电流、RPM、位置和开环控制。`mcpwm.c` 负责传统 BLDC/PWM 控制，`mcpwm_foc.c` 负责 FOC 控制，`foc_math.c` 提供 FOC 数学计算。电机状态、故障码、转速、电流、电压、温度和里程统计都通过该层维护。

## 3. 编码器与位置反馈模块

相关目录为 `encoder/`。

该模块统一管理多种位置传感器，支持 ABI、AS504x、AS5x47U、BISS-C、MT6816、PWM、Sin/Cos、TLE5012、TS5700N8501 等编码器。它为 FOC、位置 PID、转子位置显示和调试逻辑提供角度反馈。

## 4. 通信模块

相关目录为 `comm/`，主要文件包括 `commands.c`、`packet.c`、`comm_usb.c`、`comm_can.c`、`comm_usb_serial.c`。

`packet.c` 实现通用数据包收发和解析；`commands.c` 解析上位机或 VESC Tool 命令，并分发到电机控制、配置、应用和 BMS 模块；`comm_usb.c` 提供 USB CDC 通信；`comm_can.c` 提供 CAN 通信、状态广播、控制指令、BMS 数据和节点发现。

```text
USB/CAN/UART/NRF
    -> packet_process_byte()
    -> commands_process_packet()
    -> mc_interface / conf_general / app / bms
```

## 5. 应用输入模块

相关目录为 `applications/`。

该模块管理外部控制输入和用户应用逻辑，支持 PPM、ADC、UART、Nunchuk、PAS、自定义应用等。系统通过 `app_configuration` 选择启用的应用，并将油门、刹车、方向、踏频等输入转换为电机控制命令。

## 6. 配置与序列化模块

相关文件包括 `conf_general.c`、`conf_general.h`、`confgenerator.c`、`datatypes.h`、`conf_custom.c`。

该模块定义电机配置 `mc_configuration` 和应用配置 `app_configuration`，提供默认配置、读取、存储、CRC 校验和版本兼容能力。`confgenerator.c` 负责配置序列化和反序列化，用于通信传输和上位机配置。

## 7. 硬件抽象与板级适配模块

相关目录为 `hwconf/`。

该模块为不同硬件板定义 GPIO、ADC、PWM、CAN、SPI、驱动芯片、温度采样等硬件差异。构建系统根据目标板选择对应 `hw_*` 文件，通过宏和板级源文件隔离硬件差异。

## 8. 底层驱动模块

相关目录为 `driver/`。

该模块提供 EEPROM、I2C bitbang、SPI bitbang、LED PWM、舵机输入、PWM Servo、NRF、LoRa、定时器等底层驱动，为应用模块、通信模块和硬件抽象层提供设备访问能力。

## 9. IMU 与姿态模块

相关目录为 `imu/`。

该模块支持 MPU9150、ICM20948、BMI160、LSM6DS3 等 IMU，并集成 AHRS 与 Fusion 姿态估算逻辑。系统启动后会调用 `imu_reset_orientation()` 重置姿态状态。

## 10. BMS 电池管理模块

相关文件为 `bms.c`、`bms.h`。

该模块管理电池电压、电流、温度、SOC/SOH、均衡状态等数据，并通过 CAN 或命令接口上报给其他节点和上位机。

## 11. LispBM 与脚本扩展模块

相关目录为 `lispBM/`。

该模块集成 LispBM 脚本运行环境，允许用户通过脚本扩展控制逻辑。`lispif_vesc_extensions.c` 提供 VESC 专用扩展函数，脚本代码可从 Flash 中加载或擦除。

## 12. UAVCAN / libcanard 模块

相关目录为 `libcanard/`。

该模块集成 UAVCAN/CANard 协议栈，支持 ESC 状态、RPM 命令、RawCommand、节点信息、参数访问和固件更新等 UAVCAN 消息，并与 `comm_can.c` 配合实现 CAN 网络能力。

## 13. 工具与通用库模块

相关目录为 `util/`。

该模块提供二进制打包/解包、CRC 校验、数字滤波、固定内存池、数学工具、系统辅助函数、异步工作任务和 LZO 压缩支持。

## 14. 构建与测试模块

相关文件和目录包括 `Makefile`、`make/fw.mk`、`make/unittest.mk`、`tests/`。

顶层 `Makefile` 根据 `hwconf` 自动生成支持的板级目标。常用命令包括 `make fw_<board>` 构建指定板固件，`make all_fw` 构建所有板，`make all_ut_run` 运行单元测试。

# 模块关系总结

```text
main.c
 ├── hwconf / driver        初始化硬件与外设
 ├── conf_general           读取和保存系统配置
 ├── mc_interface           电机控制统一入口
 │    ├── mcpwm             BLDC/PWM 控制
 │    ├── mcpwm_foc         FOC 控制
 │    ├── foc_math          FOC 数学计算
 │    └── encoder           位置反馈
 ├── comm
 │    ├── packet            数据包协议
 │    ├── commands          命令分发
 │    ├── comm_usb          USB 通信
 │    └── comm_can          CAN 通信
 ├── applications           外部输入应用
 ├── imu                    姿态与惯性测量
 ├── bms                    电池状态管理
 ├── lispBM                 脚本扩展
 └── util                   通用工具
```

# 核心设计特点

- **分层清晰**：硬件差异集中在 `hwconf/`，业务逻辑集中在 `motor/`、`comm/`、`applications/`。
- **实时控制**：基于 ChibiOS 线程和中断，电机控制依赖 ADC/PWM/定时器实时运行。
- **统一控制接口**：外部命令、应用输入、CAN 指令最终大多汇聚到 `mc_interface_*`。
- **强配置驱动**：电机参数和应用参数通过 `mc_configuration`、`app_configuration` 控制行为。
- **多通信入口**：USB、UART、CAN、NRF、UAVCAN 均可参与控制或状态上报。
- **硬件可移植**：通过 `hwconf` 和 Makefile 目标支持大量不同控制器硬件。
