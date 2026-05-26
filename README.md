# grbl-xyonly 项目文件结构梳理（中文）

> 仓库来源：<https://github.com/shantu246/grbl-xyonly>
> 
> 本文档聚焦“项目文件结构 + 各文件作用 + 主要函数结构”。

## 1. 项目整体目录

```text
grbl-xyonly/
├─ .gitignore
├─ COPYING
├─ Makefile
├─ README.md
├─ build/
│  └─ .gitignore
├─ doc/
│  ├─ log/
│  │  ├─ commit_log_v0.7.txt
│  │  ├─ commit_log_v0.8c.txt
│  │  ├─ commit_log_v0.9g.txt
│  │  ├─ commit_log_v0.9i.txt
│  │  └─ commit_log_v0.9j.txt
│  └─ script/
│     ├─ simple_stream.py
│     └─ stream.py
└─ grbl/
   ├─ *.c / *.h（核心固件）
   ├─ cpu_map/
   │  ├─ cpu_map_atmega328p.h
   │  └─ cpu_map_atmega2560.h
   ├─ defaults/
   │  ├─ defaults_generic.h
   │  ├─ defaults_oxcnc.h
   │  ├─ defaults_shapeoko.h
   │  ├─ defaults_shapeoko2.h
   │  ├─ defaults_shapeoko3.h
   │  ├─ defaults_sherline.h
   │  ├─ defaults_simulator.h
   │  ├─ defaults_x_carve_500mm.h
   │  ├─ defaults_x_carve_1000mm.h
   │  └─ defaults_zen_toolworks_7x7.h
   └─ examples/grblUpload/
      ├─ grblUpload.ino
      └─ license.txt
```

---

## 2. 根目录与辅助目录文件说明

- `.gitignore`：Git 忽略规则。
- `COPYING`：GPLv3 许可证文本。
- `Makefile`：AVR 编译/烧录入口（`all`、`flash`、`fuse`、`clean` 等目标）。
- `README.md`：当前文档。
- `build/.gitignore`：保留构建目录但忽略构建产物。

### `doc/log/`
- `commit_log_v0.7.txt`：v0.7 变更日志。
- `commit_log_v0.8c.txt`：v0.8c 变更日志。
- `commit_log_v0.9g.txt`：v0.9g 变更日志。
- `commit_log_v0.9i.txt`：v0.9i 变更日志。
- `commit_log_v0.9j.txt`：v0.9j 变更日志。

### `doc/script/`
- `simple_stream.py`：简化版串口 G-code 流发送脚本（命令行直推）。
- `stream.py`：更完整的串口流控发送脚本（带更细致的发送/缓冲处理逻辑）。
- 函数结构：两个脚本以脚本主流程为主，无显式 `def` 函数定义。

### `grbl/examples/grblUpload/`
- `grblUpload.ino`：Arduino IDE 示例入口，用于将 Grbl 作为库上传到板子。
- `license.txt`：示例目录的 MIT 许可证文本。

---

## 3. `grbl/` 核心固件文件结构

> 说明：`*.h` 主要定义宏、结构体、状态位与对外函数声明；`*.c` 提供实现。

### 3.1 系统入口与总头文件

#### `grbl/main.c`
- 作用：固件主入口，完成初始化并进入协议主循环。
- 主要函数：
  - `main()`：初始化串口、设置、步进、系统状态；循环调用 `protocol_main_loop()`。

#### `grbl/grbl.h`
- 作用：总入口头文件，汇总所有模块头文件并定义版本号。
- 函数结构：无函数实现；负责统一 include 与全局编译上下文。

---

### 3.2 运动控制链路（G-code → 规划器 → 步进执行）

#### `grbl/gcode.h` / `grbl/gcode.c`
- 作用：G-code 语法解析与执行调度。
- 主要函数：
  - `gc_init()`：解析器状态初始化。
  - `gc_execute_line()`：执行一行 G-code。
  - `gc_sync_position()`：同步解析器位置。
  - 内部辅助：位置一致性检查等静态逻辑（含 `gc_check_same_position`）。

#### `grbl/motion_control.h` / `grbl/motion_control.c`
- 作用：高层运动指令接口（直线、圆弧、探针、回零等）。
- 主要函数：
  - `mc_line()`：直线插补入口。
  - `mc_arc()`：圆弧分段插补。
  - `mc_dwell()`：暂停（G4）。
  - `mc_homing_cycle()`：回零流程。
  - `mc_probe_cycle()`：探针流程。
  - `mc_reset()`：运动相关紧急复位。

#### `grbl/planner.h` / `grbl/planner.c`
- 作用：运动规划缓存与加减速前瞻（look-ahead）。
- 主要函数：
  - `plan_reset()`
  - `plan_buffer_line()`
  - `plan_discard_current_block()`
  - `plan_get_current_block()`
  - `plan_next_block_index()`
  - `plan_get_exec_block_exit_speed()`
  - `plan_sync_position()`
  - `plan_cycle_reinitialize()`
  - `plan_get_block_buffer_count()`
  - `plan_check_full_buffer()`
  - 内部辅助：`planner_recalculate()`、`plan_prev_block_index()` 等。

#### `grbl/stepper.h` / `grbl/stepper.c`
- 作用：步进电机底层执行器，按规划块生成 step/dir 时序。
- 主要函数：
  - `stepper_init()`
  - `st_wake_up()` / `st_go_idle()`
  - `st_generate_step_dir_invert_masks()`
  - `st_reset()`
  - `st_prep_buffer()`
  - `st_update_plan_block_parameters()`
  - `st_get_realtime_rate()`
  - 中断：`ISR(...)`（步进相关定时/执行中断入口）。

#### `grbl/protocol.h` / `grbl/protocol.c`
- 作用：协议层主循环与实时命令执行。
- 主要函数：
  - `protocol_main_loop()`
  - `protocol_execute_realtime()`
  - `protocol_buffer_synchronize()`
  - `protocol_auto_cycle_start()`
  - 内部：`protocol_execute_line()`（单行命令分发）。

---

### 3.3 系统状态、配置与持久化

#### `grbl/system.h` / `grbl/system.c`
- 作用：系统级状态机、实时控制位、系统命令（`$` 命令）处理。
- 主要函数：
  - `system_init()`
  - `system_check_safety_door_ajar()`
  - `system_execute_line()`
  - `system_execute_startup()`
  - `system_convert_axis_steps_to_mpos()`
  - `system_convert_array_steps_to_mpos()`
  - CoreXY 辅助：`system_convert_corexy_to_x_axis_steps()` / `..._y_axis_steps()`
  - 中断：`ISR(...)`（控制引脚变化中断入口）。

#### `grbl/settings.h` / `grbl/settings.c`
- 作用：参数系统（`$` 配置）读写与 EEPROM 持久化。
- 主要函数：
  - `settings_init()`
  - `settings_restore()`
  - `settings_store_global_setting()`
  - `settings_store_startup_line()` / `settings_read_startup_line()`
  - `settings_store_build_info()` / `settings_read_build_info()`
  - `settings_write_coord_data()` / `settings_read_coord_data()`
  - `get_step_pin_mask()` / `get_direction_pin_mask()` / `get_limit_pin_mask()`
  - 内部：`read_global_settings()`、`write_global_settings()`。

#### `grbl/eeprom.h` / `grbl/eeprom.c`
- 作用：EEPROM 底层读写与校验封装。
- 主要函数：
  - `eeprom_get_char()` / `eeprom_put_char()`
  - `memcpy_to_eeprom_with_checksum()`
  - `memcpy_from_eeprom_with_checksum()`

#### `grbl/config.h`
- 作用：编译期开关与功能宏配置（限位、探针、报告、步进行为等）。
- 函数结构：无函数；以宏配置为核心。

#### `grbl/defaults.h`
- 作用：选择默认机型参数模板并注入默认设置。
- 函数结构：无函数；通过宏包含具体 defaults 文件。

#### `grbl/defaults/*.h`
- 作用：各机型默认参数模板。
- 文件与用途：
  - `defaults_generic.h`：通用默认参数。
  - `defaults_oxcnc.h`：OX CNC 机型参数。
  - `defaults_shapeoko.h` / `defaults_shapeoko2.h` / `defaults_shapeoko3.h`：Shapeoko 系列参数。
  - `defaults_sherline.h`：Sherline 参数。
  - `defaults_simulator.h`：仿真器参数。
  - `defaults_x_carve_500mm.h` / `defaults_x_carve_1000mm.h`：X-Carve 参数。
  - `defaults_zen_toolworks_7x7.h`：Zen Toolworks 7x7 参数。
- 函数结构：无函数；均为默认值宏集合。

#### `grbl/cpu_map.h` 与 `grbl/cpu_map/*.h`
- 作用：芯片与引脚映射抽象层。
- 文件与用途：
  - `cpu_map.h`：总入口，根据目标 MCU 选择具体映射。
  - `cpu_map_atmega328p.h`：ATmega328P 引脚映射。
  - `cpu_map_atmega2560.h`：ATmega2560 引脚映射。
- 函数结构：无函数；均为引脚/寄存器宏定义。

---

### 3.4 外设与运行时功能模块

#### `grbl/serial.h` / `grbl/serial.c`
- 作用：串口驱动（RX/TX 环形缓冲区）。
- 主要函数：
  - `serial_init()`
  - `serial_write()`
  - `serial_read()`
  - `serial_reset_read_buffer()`
  - `serial_get_rx_buffer_count()` / `serial_get_tx_buffer_count()`
  - 中断：`ISR(...)`（串口收发中断处理）。

#### `grbl/report.h` / `grbl/report.c`
- 作用：状态、告警、参数、反馈等统一上报。
- 主要函数：
  - `report_status_message()`
  - `report_alarm_message()`
  - `report_feedback_message()`
  - `report_init_message()`
  - `report_grbl_help()`
  - `report_grbl_settings()`
  - `report_echo_line_received()`
  - `report_realtime_status()`
  - `report_probe_parameters()`
  - `report_ngc_parameters()`
  - `report_gcode_modes()`
  - `report_startup_line()`
  - `report_build_info()`

#### `grbl/print.h` / `grbl/print.c`
- 作用：字符串与数字格式化输出工具。
- 主要函数：
  - `printString()` / `printPgmString()`
  - `printInteger()`
  - `print_uint32_base10()`
  - `print_unsigned_int8()`
  - `print_uint8_base2()` / `print_uint8_base10()`
  - `printFloat()`
  - `printFloat_CoordValue()` / `printFloat_RateValue()` / `printFloat_SettingValue()`

#### `grbl/spindle_control.h` / `grbl/spindle_control.c`
- 作用：主轴控制（启停、方向、转速/PWM）。
- 主要函数：
  - `spindle_init()`
  - `spindle_run()`
  - `spindle_set_state()`
  - `spindle_stop()`

#### `grbl/coolant_control.h` / `grbl/coolant_control.c`
- 作用：冷却液控制（M7/M8/M9）。
- 主要函数：
  - `coolant_init()`
  - `coolant_stop()`
  - `coolant_set_state()`
  - `coolant_run()`

#### `grbl/limits.h` / `grbl/limits.c`
- 作用：硬限位、软限位、回零寻边。
- 主要函数：
  - `limits_init()` / `limits_disable()`
  - `limits_get_state()`
  - `limits_go_home()`
  - `limits_soft_check()`
  - 中断：`ISR(...)`（限位触发实时处理）。

#### `grbl/probe.h` / `grbl/probe.c`
- 作用：探针输入状态管理与探测监控。
- 主要函数：
  - `probe_init()`
  - `probe_configure_invert_mask()`
  - `probe_get_state()`
  - `probe_state_monitor()`

#### `grbl/nuts_bolts.h` / `grbl/nuts_bolts.c`
- 作用：公共工具函数（解析、延时、数学工具）。
- 主要函数：
  - `read_float()`
  - `delay_ms()` / `delay_us()`
  - `hypot_f()`

---

## 4. 工程结构总体说明

这个项目是典型的“**嵌入式实时控制固件分层架构**”：

1. **输入层**：`serial` + `protocol` 接收与调度串口/G-code 命令。  
2. **语义层**：`gcode` 把文本命令解析为标准运动/控制语义。  
3. **规划层**：`motion_control` + `planner` 将语义运动转换为可执行轨迹并做前瞻加减速。  
4. **执行层**：`stepper` 根据规划结果以中断驱动输出 step/dir 脉冲。  
5. **设备控制层**：`spindle_control`、`coolant_control`、`limits`、`probe` 负责外设与安全联锁。  
6. **系统与持久化层**：`system`、`settings`、`eeprom` 负责状态机、配置管理和掉电保存。  
7. **配置抽象层**：`config`、`defaults`、`cpu_map` 将编译期行为、机型参数、引脚映射解耦。  
8. **输出层**：`print`、`report` 统一对上位机反馈状态与结果。

整体上，项目以 `main.c` 为入口，通过“协议驱动 + 运动规划 + 中断执行”实现 CNC 实时控制；文件边界清晰，模块职责明确，便于在不同机型（defaults）和 MCU（cpu_map）间复用与裁剪。

## 5.修改要点
M33 S1  ; 将 D9 输出置高
M33 S0  ; 将 D9 输出置低
M44 S1  ; 将 D10 输出置高
M44 S0  ; 将 D10 输出置低

注释了一些Z轴相关代码

D4: Z 轴步进脉冲（Z step）— 在引脚映射中为 Z_STEP_BIT（Uno 数字引脚 D4）。证据: cpu_map_atmega328p.h:20-40。步进逻辑使用处: stepper.c:280-320。

D7: Z 轴方向（Z direction）— 在引脚映射中为 Z_DIRECTION_BIT（Uno 数字引脚 D7）。证据: cpu_map_atmega328p.h:36-60。使用处: stepper.c:380-394。

D9: X 轴限位开关输入（X limit）— 在映射中为 X_LIMIT_BIT（Uno 数字引脚 D9，位于 LIMIT_PORT/PCINT 组）。证据: cpu_map_atmega328p.h:48-72。限位初始化/中断使用处: limits.c:1-40。

D10: Y 轴限位开关输入（Y limit）— 在映射中为 Y_LIMIT_BIT（Uno 数字引脚 D10）。证据: cpu_map_atmega328p.h:48-72。限位处理处: limits.c:1-40。
