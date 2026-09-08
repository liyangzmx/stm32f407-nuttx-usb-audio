# NuttX USB Audio Device

## 1. 项目概述

本项目将 `/opt/coding/stm32/STM32F407-DISC-Audio` 中基于 FreeRTOS、STM32
HAL 和 ST USB Device Library 的 USB Audio 播放固件迁移到 Apache NuttX。

目标硬件为 STM32F407G-DISC1/STM32F4DISCOVERY。开发板通过 USB OTG FS
连接电脑并枚举为 USB Audio Class 1.0 扬声器，接收电脑输出的 PCM 音频，再通过
I2S3 和 DMA 发送到板载 CS43L22 Codec。

当前支持的固定音频格式：

| 项目 | 参数 |
|---|---|
| USB 类 | USB Audio Class 1.0 |
| 方向 | Host → Device，扬声器播放 |
| 采样率 | 48 kHz |
| 采样位宽 | 16 bit |
| 声道数 | 2，立体声 |
| USB 传输 | Full-Speed Isochronous Adaptive OUT |
| 音频端点 | EP1 OUT |
| 每帧数据 | 192 字节/毫秒 |
| Codec | CS43L22 |
| Codec 控制 | I2C1 |
| PCM 输出 | I2S3 + DMA |

## 2. 源工程分析

原 FreeRTOS 工程的主要职责分布如下：

| 原工程模块 | 作用 | NuttX 对应实现 |
|---|---|---|
| `Core/Src/main.c` | 时钟、GPIO、I2C1、I2S3、DMA 和 RTOS 初始化 | STM32F4Discovery BSP、NuttX 驱动配置和 board bring-up |
| `USB_DEVICE/App/usb_device.c` | 初始化 ST USB Device Stack 并注册 Audio 类 | NuttX USB device controller 与 `uac1.c` |
| `USB_DEVICE/App/usbd_audio_if.c` | USB 音频回调、PCM 缓冲和 Codec 播放控制 | `/dev/uac1` 与 `uacplay` |
| `Middlewares/ST/.../usbd_audio.c` | UAC1 描述符、控制请求、等时端点处理 | NuttX UAC1 gadget 驱动 |
| BSP Audio/CS43L22 | Codec 寄存器配置和音频输出 | NuttX CS43L22 lower-half 与 Audio upper-half |

原工程的关键硬件参数为：I2S3 Master TX、Philips I2S 标准、16-bit 数据、
48 kHz、MCLK 输出、PLL I2S 时钟源。NuttX 版本保持了相同的音频链路和 USB
数据格式。

## 3. NuttX 软件架构

```text
┌──────────────────────────────┐
│ PC / macOS / Linux / Windows │
│ USB Audio Host               │
└──────────────┬───────────────┘
               │ UAC1 Isochronous OUT
               │ 192 bytes every 1 ms
               ▼
┌──────────────────────────────┐
│ STM32 OTG FS Device Driver   │
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│ UAC1 Gadget Driver           │
│ /dev/uac1                    │
│ - USB descriptors            │
│ - alternate interface        │
│ - mute/volume control        │
│ - USB receive request queue  │
└──────────────┬───────────────┘
               │ PCM byte stream
               ▼
┌──────────────────────────────┐
│ uacplay                      │
│ - buffer prefill             │
│ - reconnect loop             │
│ - volume mapping             │
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│ NuttX Audio Upper-Half       │
│ /dev/audio/pcm1              │
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│ CS43L22 Lower-Half           │
│ I2C1 control + I2S3 DMA      │
└──────────────┬───────────────┘
               ▼
          Headphone output
```

### 3.1 UAC1 gadget 驱动

驱动位置：`nuttx/drivers/usbdev/uac1.c`。

驱动包含以下功能：

- USB Device、Configuration、Interface、Audio Class 和 Endpoint 描述符；
- Audio Control Interface 和 Audio Streaming Interface；
- Streaming Interface alternate setting 0/1；
- 48 kHz、16-bit、双声道 Type I PCM Format 描述符；
- Feature Unit 主通道 Mute 和 Volume 控制；
- `GET_CUR`、`GET_MIN`、`GET_MAX`、`GET_RES` 和 `SET_CUR` 请求；
- 多个预提交 USB OUT request，降低调度延迟导致的丢包风险；
- 将收到的 PCM 数据通过 `/dev/uac1` 暴露给应用；
- 在断开连接或退出 streaming alternate setting 时唤醒阻塞读取者。

公开接口定义在 `nuttx/include/nuttx/usb/uac1.h`。应用可通过
`UAC1IOC_GETSTATUS` 获取配置状态、streaming 状态、静音状态和 USB 音量。

### 3.2 uacplay 桥接应用

应用位置：`apps/system/uacplay/uacplay_main.c`。

启动后执行以下流程：

1. 打开 `/dev/uac1`；
2. 初始化 `/dev/audio/pcm1`，配置为 48 kHz、16-bit、双声道；
3. 等待 USB Host 选择 Audio Streaming alternate setting 1；
4. 预填充四个 1536 字节音频缓冲，总启动弹性约 32 ms；
5. 启动 CS43L22 播放；
6. 收到 NuttX Audio dequeue 消息后，从 USB 数据流重新填充该缓冲；
7. 把 UAC1 的 -63 dB～0 dB 音量映射到 NuttX 的 0～1000 音量范围；
8. USB 断开后停止当前 Audio session，等待重新连接并自动恢复。

### 3.3 板级初始化

`stm32_bringup()` 先注册 CS43L22 音频设备，再初始化 UAC1 gadget。设备节点为：

```text
/dev/audio/pcm1   CS43L22 PCM 输出
/dev/uac1         USB Host PCM 输入
```

配置使用 `CONFIG_BOARD_LATE_INITIALIZE`，因此进入 `uacplay_main` 前硬件和设备
节点已经完成初始化。

## 4. USB 描述符结构

UAC1 Configuration Descriptor 总长度为 110 字节，包含：

```text
Configuration
├── Interface 0: Audio Control
│   ├── Class-Specific AC Header
│   ├── Input Terminal: USB Streaming
│   ├── Feature Unit: Mute + Volume
│   └── Output Terminal: Speaker
├── Interface 1, Alternate 0: no endpoint
└── Interface 1, Alternate 1: active streaming
    ├── AS General Descriptor
    ├── Type I PCM Format Descriptor
    ├── EP1 OUT, Isochronous Adaptive, 192 bytes, 1 ms
    └── Class-Specific Audio Endpoint Descriptor
```

PCM 带宽计算：

```text
48000 samples/s × 2 channels × 2 bytes/sample = 192000 bytes/s
192000 bytes/s ÷ 1000 USB frames/s = 192 bytes/frame
```

## 5. 目录与改动

```text
/opt/coding/stm32/nuttx_audio/
├── README.md
├── Nuttx_USB_Audio_Device.md
├── nuttx/
│   ├── drivers/usbdev/uac1.c
│   ├── include/nuttx/usb/uac1.h
│   ├── drivers/usbdev/Kconfig
│   ├── drivers/usbdev/Make.defs
│   ├── drivers/usbdev/CMakeLists.txt
│   ├── arch/arm/src/common/stm32/stm32_otgfsdev_m3m4_v1.c
│   └── boards/arm/stm32f4/stm32f4discovery/
│       ├── configs/audio-usb/defconfig
│       └── src/stm32_bringup.c
└── apps/
    └── system/uacplay/
        ├── Kconfig
        ├── Make.defs
        ├── Makefile
        ├── CMakeLists.txt
        └── uacplay_main.c
```

NuttX 和 NuttX Apps 均基于 `nuttx-13.0.0`。GitHub 精简快照使用 `main`
分支；本地仍保留带 Apache 浅历史的 `nuttx-audio` 开发分支。

NuttX 13.0.0 的 STM32 OTG FS 驱动在启用
`CONFIG_USBDEV_ISOCHRONOUS` 后有缺失局部变量声明的问题。本项目在
`stm32_otgfsdev_m3m4_v1.c` 中进行了最小修复，使 isochronous 路径能够编译。

## 6. 关键 Kconfig 配置

板级配置位于：

```text
nuttx/boards/arm/stm32f4/stm32f4discovery/configs/audio-usb/defconfig
```

核心选项：

```text
CONFIG_STM32_OTGFS=y
CONFIG_USBDEV=y
CONFIG_USBDEV_ISOCHRONOUS=y
CONFIG_UAC1=y
CONFIG_UAC1_NRDREQS=16

CONFIG_AUDIO=y
CONFIG_AUDIO_CS43L22=y
CONFIG_STM32_I2C1=y
CONFIG_STM32_I2S3=y
CONFIG_STM32_I2S3_TX=y
CONFIG_STM32_I2S_MCK=y

CONFIG_CS43L22_BUFFER_SIZE=1536
CONFIG_CS43L22_NUM_BUFFERS=4

CONFIG_SYSTEM_UACPLAY=y
CONFIG_INIT_ENTRYPOINT="uacplay_main"
```

USB 默认标识：

```text
VID:          0x0483
PID:          0x5740
Manufacturer: NuttX
Product:      STM32F407 NuttX Audio
Serial:       F407AUDIO
```

如果产品化使用，应申请并替换合法 VID/PID，同时为每台设备提供唯一序列号。

## 7. 构建方法

### 7.1 环境要求

- Arm GNU Toolchain，命令前缀为 `arm-none-eabi-`；
- GNU Make；
- NuttX 与 Apps 保持当前同级目录布局。

### 7.2 从干净配置构建

```sh
cd /opt/coding/stm32/nuttx_audio/nuttx
make distclean
./tools/configure.sh -l stm32f4discovery:audio-usb
make -j4
```

输出文件：

```text
nuttx/nuttx       ELF 文件
nuttx/nuttx.bin   原始二进制
nuttx/nuttx.hex   Intel HEX
nuttx/nuttx.map   Linker map
```

当前验证构建的资源占用：

```text
text     68736 bytes
data      1572 bytes
bss       4904 bytes
total    75212 bytes

Flash image: 70308 bytes
Static SRAM: 6488 bytes
```

当前 `nuttx.bin` SHA-256：

```text
d8e6a2a2b9dc0d11463c1ad77be23ae48e01648965ec4e69ad9cced42b356edf
```

## 8. 烧录

使用 STM32CubeProgrammer：

```sh
cd /opt/coding/stm32/nuttx_audio/nuttx
STM32_Programmer_CLI -c port=SWD -w nuttx.bin 0x08000000 -v -rst
```

使用 `st-flash`：

```sh
cd /opt/coding/stm32/nuttx_audio/nuttx
st-flash write nuttx.bin 0x08000000
```

也可以使用 OpenOCD、J-Link 或其他支持 STM32F407 的 SWD 工具。

## 9. 使用方法

1. 烧录 `nuttx.bin`；
2. 给开发板供电；
3. 将电脑连接到开发板 USB OTG FS 接口，而不是仅连接 ST-LINK USB 接口；
4. 在操作系统声音设置中选择 `STM32F407 NuttX Audio`；
5. 播放 48 kHz 音频；
6. 从板载耳机接口检查输出；
7. 如需日志，通过 USART2 连接串口，参数为 115200 8N1。

典型日志：

```text
uacplay: waiting for USB host audio
uacplay: streaming 48000 Hz, stereo, 16-bit PCM
uacplay: stream stopped (...)
```

## 10. 上板验证建议

### 10.1 USB 枚举

Linux：

```sh
lsusb
lsusb -v -d 0483:5740
aplay -l
```

macOS 可在“系统信息 → USB”和“音频 MIDI 设置”中检查设备。Windows 可在
设备管理器和声音设置中检查设备，应使用系统自带 USB Audio Class 驱动，无需
自定义驱动。

检查项：

- VID/PID 与字符串正确；
- Configuration 总长度为 110；
- Audio Control 和 Audio Streaming 两个接口存在；
- alternate setting 1 包含 EP1 OUT；
- 主机只报告 48 kHz、16-bit、双声道格式。

### 10.2 播放测试

建议依次执行：

1. 播放 1 kHz 正弦波，检查左右声道和幅度；
2. 播放音乐 5～10 分钟，检查爆音、丢帧和缓冲下溢；
3. 调整系统音量并测试静音；
4. 播放过程中拔插 USB；
5. 停止、重新开始播放；
6. 长时间播放，重点观察 USB 与 I2S 独立时钟造成的漂移。

Linux 示例：

```sh
speaker-test -D default -c 2 -r 48000 -F S16_LE -t sine
```

### 10.3 USB 抓包

如果枚举失败或播放异常，可使用 Linux usbmon + Wireshark 检查：

- `SET_CONFIGURATION(1)` 是否成功；
- Host 是否发送 `SET_INTERFACE(interface=1, alt=1)`；
- EP1 OUT 是否每毫秒收到约 192 字节；
- Feature Unit 的 Mute/Volume 请求是否返回成功；
- 播放停止时是否切回 alternate setting 0。

## 11. 已完成验证与限制

已经完成：

- NuttX 与 NuttX Apps 13.0.0 配置；
- 从 `make distclean` 开始的完整编译和链接；
- `nuttx.bin`、`nuttx.hex` 和 ELF 产物生成；
- UAC1 driver、公共头文件和 uacplay 的 NuttX 风格检查；
- Git whitespace/diff 检查；
- 最终 ELF 中确认包含 `stm32_bringup`、CS43L22 初始化、
  `usbdev_uac1_initialize` 和 `uacplay_main`。

尚未完成：

- 真实 STM32F407 Discovery 开发板上的 USB 枚举测试；
- 实际耳机输出验证；
- 不同操作系统的兼容性测试；
- USB SOF 时钟与 I2S PLL 长时间偏差测试；
- USB suspend/resume 和压力测试。

因此，当前状态是“固件实现和构建已完成，上板验证待执行”。如果长时间播放出现
周期性爆音，应优先测量 USB 接收缓冲深度和 I2S 实际采样率，再决定增加样本
丢弃/复制补偿，或改为异步端点加 feedback endpoint。

## 12. 后续可扩展项

- 支持 44.1 kHz、96 kHz 或多种采样率；
- 支持 24-bit PCM；
- 增加异步 OUT endpoint 和显式 feedback endpoint；
- 增加 USB suspend/resume 电源管理；
- 使用 STM32 UID 动态生成唯一 USB 序列号；
- 增加音频缓冲水位、丢包和 underrun 统计接口；
- 增加 NSH 调试配置和运行时诊断命令；
- 为 UAC1 描述符和控制请求增加 host-side 自动化测试。
