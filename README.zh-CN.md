# stm32-vserprog:

## 基于 STM32 MCU 与 USB CDC 协议的 flashrom serprog 编程器。

**此项目取代了先前使用意法半导体专有固件库的 [`serprog-stm32vcp`](https://github.com/dword1511/serprog-stm32vcp) 项目。大部分功能相同。**

------

### 特性

- **完全开源**：现使用 [`libopencm3`](https://github.com/libopencm3/libopencm3) 替代意法半导体的专有固件库。
- **经济实惠且简单的硬件**：
  - 一片 *STM32F103C8T6* 微控制器、一个 8MHz 晶振、一片 3.3V *1117* 低压差稳压器、若干 *0805* 封装的电容、电阻和 LED，配合专用的 PCB（可在 OSH Park 获得：[V2](https://oshpark.com/shared_projects/08Rj6sSm) 或 [V3](https://oshpark.com/shared_projects/vKn08YZG)）
  - 也可以使用某些通用的 *STM32* 开发板，只需在 "boards" 文件夹下添加一个新的头文件，为 USB D+ 上拉（仅限 *STM32F1*）和 LED 分配正确的 GPIO。
  - 硬件 USB 2.0 全速接口，高效的虚拟 COM 口，使用 USB CDC 协议，可在任意波特率下工作，省去了 USB 转 UART 桥接器及其带来的各种烦恼。
  - **讽刺的是，你仍然需要购买或借用 USB 转 UART 桥接器（注意是 TTL 电平，不是 RS-232）来给编程器本身烧录程序**，除非你使用具有 USB ISP 功能的 *STM32F0x2* 器件，或带有内置 *ST-Link* 的开发板（参见 @FabianInostroza 的复刻分支）。
- **硬件全双工 SPI 配合 DMA**，提供多种时钟速度（默认选择最接近但低于 10MHz 的那一档），例如在 STM32F103 目标上：
  - 36MHz
  - 18MHz
  - 9MHz *(默认)*
  - 4.5MHz
  - 2.25MHz
  - 1.125MHz
  - 562.5kHz
  - 281.25kHz
- **2 个状态 LED（由 1 个 GPIO 控制）**：
  - 操作期间“忙”灯亮起（对于只有一个 LED 的板子，逻辑相反）。
  - 不同操作对应不同闪烁模式：
    忙灯频繁亮起 = 读取，两灯同时亮 = 写入，交替闪烁 = 擦除。
  - 如果发生硬件错误（hard fault）或不可屏蔽中断（NMI），“忙”灯会单独闪烁。请拔掉并重新插入编程器。若情况依旧，请考虑[提交新的 issue](https://github.com/dword1511/stm32-vserprog/issues)。
- **`flashrom` “serprog”协议**：
  - 在所有主流平台上均可运行（参见 [`flashrom` wiki](http://www.flashrom.org/Flashrom)）。
  - Linux 下无需驱动。Windows 下可能需要安装 ST 的 VCP 驱动（或直接修改其他 CDC 驱动的 `.inf` 文件，尚未证实）。
- **固件除底层 USB 处理外完全无中断** —— 稳定可靠，易于调试和修改。
- **速度快**：
  - 读取速度在 36MHz SPI 下最高可达 850KiB/s。
  - 读取一片 *EN25Q64* 仅需 11 秒，写入（编程）耗时 85 秒（均包含 `flashrom` 的校准时间）。
  - 远比许多基于 *CH341* 的商业 USB 编程器更快，尤其是读取速度。
- 支持 25 和 26 系列 SPI 闪存芯片。
  **45 系列闪存芯片引脚排列不同。因此，如果使用上述 V2 版本 PCB，你需要一个转接座。**

------

### 安装

1. 安装 `stm32flash` 和 `gcc-arm-none-eabi` 工具链。

Debian 系统上，只需执行：

bash

```
sudo apt-get install stm32flash gcc-arm-none-eabi
```



注意：某些系统可能需要手动安装 `newlib`（例如 `libnewlib-arm-none-eabi`）。参见 [#27](https://github.com/dword1511/stm32-vserprog/issues/27)。

1. 克隆仓库并编译。

直接输入以下命令（请根据板子名称相应修改，详情可查看 `Makefile` 头部或直接输入 `make`）：

bash

```
git clone --recurse-submodules https://github.com/dword1511/stm32-vserprog.git
make BOARD=stm32-vserprog-v2
```



1. 烧录固件。

先将 USB 转 UART 桥接器连接到电脑。然后用导线将桥接器的 *TXD* 和 *RXD* 连接到你的 *STM32* 板子。
对于上面提到的 V2 或 V3 板子，将 *BOOT0*（板上丝印为 *BT0*）连接到 USB 转 UART 桥接器的 3.3V 输出端。
对于其他板子，找到 *BOOT0* 跳线或 ISP 开关，将其置为高电平或开启状态。
在某些板子上，你还需要确保 *BOOT1* 被拉低。
接着，通过 USB 将 *STM32* 板连接到电脑以供电。
最后，输入：

bash

```
make BOARD=stm32-vserprog-v2 flash-uart
```



1. 大功告成！

把 USB 转 UART 桥接器扔到一边，尽情享用吧。
在复位或重新插拔板子之前，别忘了把 *BOOT0* 拉低。

------

### 使用方法

以下均基于 Linux 平台，且编程器在系统中显示为 `/dev/ttyACM0`。

1. **读取闪存芯片**：

- 将 25 系列 SPI 闪存连接到板子上的 DIP-8 插座，或按照原理图连接。
- 用 USB 线将板子连接到电脑。
- 输入：

bash

```
flashrom -p serprog:dev=/dev/ttyACM0:4000000 -r 保存文件.bin
```



- 有时 `flashrom` 会要求你指定芯片型号，需要添加类似这样的参数：

bash

```
-c SST25VF040B
```



这是因为有时仅凭命令码难以区分具有不同时序要求的芯片，而目前 `flashrom` 不会通过 `RDID` 命令去读取它。

1. **擦除闪存芯片**：

- 写入操作会自动执行擦除。不过，如果你只是想要一片空白的芯片，则需要手动擦除。
- 输入：

bash

```
flashrom -p serprog:dev=/dev/ttyACM0:4000000 -E
```



- 对于某些芯片（如 *MX25L6445E*），在使用旧版本 `flashrom` 时首次擦除可能会失败，再执行一次即可。似乎是第一个区块需要更长的擦除等待时间。
- 擦除后会自动校验内容是否全部为空。
- 整个过程可能需要几分钟。

1. **写入闪存芯片**：

- 输入：

bash

```
flashrom -p serprog:dev=/dev/ttyACM0:4000000 -w 待写入文件.bin
```



- 会检查芯片内容，非空的区块会被自动擦除。
- 写入后会自动校验镜像。
- 整个过程可能需要几分钟。

------

### 遇到问题？

1. 如果你遇到类似 "`Error: Cannot open serial port: Device or resource busy`" 的错误，请尝试停止或卸载 `ModemManager`。
2. 检查你的接线和 `flashrom` 版本。如果在面包板或原型板上操作，不要忘记给闪存芯片本身供电。
3. 如果你确认是编程器固件的 bug，请[提交新的 issue](https://github.com/dword1511/stm32-vserprog/issues)并提供详细信息，例如你使用的板子以及 `flashrom` 的输出。感谢你的反馈。
