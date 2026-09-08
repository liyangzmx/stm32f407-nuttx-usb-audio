# STM32F407 Discovery NuttX USB Audio

这是对 `STM32F407-DISC-Audio` FreeRTOS 固件的 NuttX 实现。目标板仍为
STM32F407G-DISC1/STM32F4DISCOVERY，电脑把开发板识别为 USB Audio Class
1.0 扬声器，音频最终由板载 CS43L22 输出。

详细设计、构建和验证说明见 `Nuttx_USB_Audio_Device.md`。

## 功能范围

- USB Full-Speed Audio Class 1.0 播放设备
- 固定格式：48 kHz、16-bit、双声道 PCM
- Isochronous adaptive OUT，EP1，每个 USB 帧 192 字节
- 支持主通道静音和音量控制（-63 dB 到 0 dB）
- NuttX 字符设备 `/dev/uac1` 接收 USB PCM
- `uacplay` 把 PCM 桥接到 `/dev/audio/pcm1`
- I2S3 + DMA 驱动板载 CS43L22，I2C1 负责编解码器控制

原工程使用 ST USB Device Audio 类、FreeRTOS 队列/任务、HAL I2S DMA 和
BSP Audio 驱动。本实现把这些职责分别放到 NuttX USB gadget、字符设备、
NuttX Audio upper-half、CS43L22 lower-half 和 `uacplay` 应用中。

```text
PC USB audio host
        |
        | UAC1, 192-byte packet every 1 ms
        v
STM32 OTG FS -> /dev/uac1 -> uacplay -> /dev/audio/pcm1
                                             |
                                             v
                                      I2S3 DMA -> CS43L22
```

## 目录

- `nuttx/`：Apache NuttX 13.0.0，包含 UAC1 gadget、板级初始化和
  `stm32f4discovery:audio-usb` 配置
- `apps/`：Apache NuttX Apps 13.0.0，包含 `system/uacplay`
- `nuttx/nuttx.bin`：当前已构建的可烧录固件
- `nuttx/nuttx.hex`：Intel HEX 格式固件

GitHub 仓库采用总入口加两个代码 submodule 的形式，代码仓库 `main` 分支为
精简快照，不包含不必要的完整 Apache 提交历史。

主要新增代码：

- `nuttx/drivers/usbdev/uac1.c`
- `nuttx/include/nuttx/usb/uac1.h`
- `apps/system/uacplay/uacplay_main.c`
- `nuttx/boards/arm/stm32f4/stm32f4discovery/configs/audio-usb/defconfig`

## 构建

需要 Arm GNU Toolchain（命令前缀 `arm-none-eabi-`）：

```sh
cd /opt/coding/stm32/nuttx_audio/nuttx
make distclean
./tools/configure.sh -l stm32f4discovery:audio-usb
make -j4
```

当前构建结果：

```text
text    data    bss     total
68736   1572    4904    75212 bytes
```

## 烧录与使用

通过板载 ST-LINK 烧录二进制：

```sh
STM32_Programmer_CLI -c port=SWD -w nuttx.bin 0x08000000 -v -rst
```

也可以使用 OpenOCD 或 `st-flash write nuttx.bin 0x08000000`。烧录后将电脑连接到
开发板的 USB OTG FS 接口；系统应枚举出 `STM32F407 NuttX Audio`。选择它作为
音频输出后，固件会自动等待数据、预填充 32 ms 音频缓冲并开始播放。USART2
控制台配置为 115200 8N1，可查看 `uacplay` 启停日志。

## 验证状态

配置、依赖关系和完整固件链接已经验证通过。NuttX 13.0.0 的 STM32 OTG FS
驱动在启用 isochronous 支持时缺少几个局部变量声明，本工程已做最小修复。
目前尚未在实际开发板上验证 USB 枚举、连续播放和时钟漂移；首次上板建议依次
检查枚举描述符、短时播放、长时间播放、静音/音量和拔插重连。
