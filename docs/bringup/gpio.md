# GPIO Bring-up 文档

## 概述

在 ESP32-P4X-Function-EV-Board V1.6 上完成 GPIO4 输出引脚的 bring-up，
通过 NuttX GPIO 子系统注册为 `/dev/gpio0`，可在 NSH 中控制高低电平。

## 硬件核对

| 项目 | 结论 | 依据 |
| --- | --- | --- |
| 目标GPIO | GPIO4 | 原理图 V1.7 第3页 J1 Pin Header For GPIOs |
| J1位置 | Pin 18（右列） | 原理图 CON20X2，CNN_GPIO4 连接到 Pin 18 |
| 是否被占用 | 否 | 网络名 CNN_GPIO4，无板载外设连接 |
| 输出电平 | 3.3V | ESP32-P4 IO 域电压 |
| GND位置 | Pin 39（左列底） | 原理图 J1 左列最后一脚接 GND |
| 测量方法 | 万用表/LED | GPIO4(Pin18) -> 330R -> LED -> GND(Pin39) |

## 代码修改清单

| 文件 | 操作 | 说明 |
| --- | --- | --- |
| `board/contest_board/include/board.h` | 修改 | BOARD_NGPIOOUT=1, BOARD_NGPIOINT=0 |
| `board/contest_board/src/esp32p4_gpio.c` | 新建 | GPIO4 输出驱动，注册 /dev/gpio0 |
| `board/contest_board/src/esp32p4_bringup.c` | 修改 | 添加 esp_gpio_init() 调用 |
| `board/contest_board/src/CMakeLists.txt` | 修改 | 添加 esp32p4_gpio.c 条件编译 |
| `board/contest_board/src/Makefile` | 修改 | 添加 esp32p4_gpio.c 条件编译 |
| `board/contest_board/configs/nsh/defconfig` | 修改 | CONFIG_DEV_GPIO=y, CONFIG_EXAMPLES_GPIO=y |

## 技术要点

### 调用链

```
系统启动 -> esp_bringup() -> esp_gpio_init()
  -> esp_gpio_matrix_out(4, SIG_GPIO_OUT_IDX, 0, 0)  // 连接到 GPIO 输出信号
  -> esp_configgpio(4, INPUT | OUTPUT)                 // 配置为 GPIO 模式
  -> esp_gpiowrite(4, 0)                               // 初始低电平
  -> gpio_pin_register(&dev, 0)                        // 注册为 /dev/gpio0
```

### 关键配置说明

- `esp_configgpio(pin, INPUT | OUTPUT)` 不带 FUNCTION 标志，让引脚选择 PIN_FUNC_GPIO 模式
- `esp_gpio_matrix_out()` 将引脚路由到简单 GPIO 输出信号（SIG_GPIO_OUT_IDX）
- 不使用 `OUTPUT_FUNCTION_1`，那会将引脚绑定到外设功能而非纯 GPIO
- `gpio_pin_register()` 返回值需检查，失败时返回负 errno

## NSH 操作命令

```bash
# 查看设备是否存在
ls /dev

# 输出高电平 (3.3V)
gpio -o 1 /dev/gpio0

# 输出低电平 (0V)
gpio -o 0 /dev/gpio0

# 查看 gpio 命令帮助
gpio -h
```

## 复位后默认状态

- GPIO4 初始化为低电平 (0V)
- 复位后 `/dev/gpio0` 正常存在
- NSH 功能不受影响 (help, free, ps 正常)

## 复现步骤

以下命令均相对于 openvela 工程根目录（即 `repo sync` 后的顶层目录）执行。

### 1. 准备 ESP HAL

ESP32-P4 构建依赖固定版本的 `esp-hal-3rdparty`，需先执行脚本克隆并打补丁：

```bash
cd contest2026_288_Bugyindudadui
bash board/contest_board/tools/prepare_esp_hal.sh
```

固定 HAL commit：`b90b1837cb5ad24747deb4c895246037cc206ce5`

### 2. 清除旧构建产物（distclean）

如果之前构建过其他配置，先执行 distclean：

```bash
cd nuttx
make distclean
cd ..
```

### 3. 构建

```bash
PATH="$HOME/.local/bin:$PATH" ./build.sh vendor/openvela/boards/contest2026_288_board/configs/nsh
```

构建成功标志：输出末尾包含 `Generated: nuttx.bin`。

构建产物：`nuttx/nuttx.bin`（ESP32-P4 Simple Boot RAM image，烧录偏移 `0x2000`）。

### 4. 烧录

```bash
sudo chmod 666 /dev/ttyACM0
PATH="$HOME/.local/bin:$PATH" esptool --chip esp32p4 --port /dev/ttyACM0 --baud 921600 \
  write_flash 0x2000 nuttx/nuttx.bin
```

烧录成功标志：`Hash of data verified.`

### 5. 串口验证

```bash
picocom -b 115200 --noreset /dev/ttyACM0
```

## 当前验证状态

### 已验证

- `/dev/gpio0` 设备节点注册成功
- NSH `gpio -o 1 /dev/gpio0` 写入后 GPIO 输入路径回读 Verify=1
- NSH `gpio -o 0 /dev/gpio0` 写入后 GPIO 输入路径回读 Verify=0
- 连续切换 4 次均回读正确
- 复位后设备恢复正常，NSH 功能（help、free、ps）不受影响

### 待验证

- J1 Pin 18 (GPIO4) 的实际 0V/3.3V 物理电平输出（需万用表或逻辑分析仪测量）
- 外接 LED 可视化验证

> **注意**：当前 `Verify=0/1` 仅代表 GPIO 寄存器输入路径的回读值，
> 不能作为 J1 Pin 18 已实际输出对应电压的物理证据。
> 物理测量条件具备后将补充接线照片和测量结果。

## 分支与提交

- 分支：`feat/gpio-bringup`
- 基线 commit：`e012c93`
