## 基础 SPI

SPI（Serial Peripheral Interface，串行外围设备接口）最早由摩托罗拉（Motorola）提出的一种高速、同步、全双工、主从式串行通信协议，常用于 MCU 与 Flash、ADC、DAC、显示屏、传感器等外设之间进行板级数据交换。

早期采用四线制设计，包括同步时钟线（SCLK）、主输出从输入线（MOSI）、主输入从输出线（MISO）和片选线（CS/SS）：

| 信号线    | 全称                     | 功能                          |
| --------- | ------------------------ | ---------------------------- |
| **SCLK**  | Serial Clock             | 主设备生成的同步时钟信号      |
| **MOSI**  | Master Out Slave In      | 主设备数据输出/从设备数据输入 |
| **MISO**  | Master In Slave Out      | 主设备数据输入/从设备数据输出 |
| **CS/SS** | Chip Select/Slave Select | 从设备片选信号（低电平有效）  |

在 IP 实现上，主从设备各包含一个移位寄存器，当时钟信号触发时，数据在寄存器间同步传递，实现全双工通信。通信架构一般就是*单主多从*，主设备通过*片选线*选择与哪个从设备通信。

## 多线 SPI

可以计算一下，如果使用单线SPI，在50MHz时钟频率下，理论最大数据传输速率为：

```
数据传输速率 = 时钟频率 × 数据线数量
            = 50 MHz × 1 bit
            = 50 Mbps
            = 6.25 MB/s
```

为了提升数据传输速率，SPI协议在2000年代引入了多线模式，最常见的是Dual SPI（双线）和Quad SPI（四线）：

| 模式                | 数据线条数    | 传输方式 | 吞吐量提升 | 通信模式 |
| ------------------- | ------------- | -------- | ---------- | -------- |
| **Standard SPI**    | 1 (MOSI+MISO) | 全双工   | 1×基准     | 全双工   |
| **Dual SPI**        | 2 (IO0+IO1)   | 半双工   | 2×         | 半双工   |
| **Quad SPI (QSPI)** | 4 (IO0-IO3)   | 半双工   | 4×         | 半双工   |

现在的 IP 实现通常支持多种模式，主设备通过配置寄存器选择使用单线、双线或四线模式进行通信，以适应不同的性能需求和兼容性要求。

可以看到 **Dual SPI** 和 **Quad SPI** 变成了半双工的通信模式，在这两个模式下，原先的 MOSI 和 MISO 线被重新定义为 IO0 和 IO1（Dual SPI），以及 IO0-IO3（Quad SPI），数据传输时只能单向进行，但通过增加数据线数量，显著提升了数据传输速率。

## SPI 配置工作流

下面是 STM32F1 系列 MCU 中 SPI 配置的示例结构体定义：
```c
typedef struct {
  uint32_t Mode;                // SPI工作模式（主/从）
  uint32_t Direction;           // 数据传输方向（全双工/半双工）
  uint32_t DataSize;            // 数据帧长度（4-16位）
  uint32_t CLKPolarity;         // 时钟极性（CPOL）
  uint32_t CLKPhase;            // 时钟相位（CPHA）
  uint32_t NSS;                 // 片选管理（软件/硬件）
  uint32_t BaudRatePrescaler;   // 波特率预分频器
  uint32_t FirstBit;            // 数据传输顺序（MSB/LSB）
  uint32_t TIMode;              // TI模式使能
  uint32_t CRCCalculation;      // CRC计算使能
  uint32_t CRCPolynomial;       // CRC多项式
} SPI_InitTypeDef;
```

各个厂商的 SPI 配置流程大致相似，通常包括以下步骤：

1. **引脚配置**：将相关引脚配置为 SPI 功能模式，通常通过 GPIO 模块的复用功能实现。
2. **时钟配置**：设置 SPI 时钟频率，通常通过时钟控制模块配置分频器来实现。
3. **模式设置**：选择 SPI 模式（Standard/Dual/Quad）和通信模式（全双工/半双工）。
4. **数据格式配置**：设置数据帧长度、时钟极性和相位等参数。
5. **启用 SPI**：通过控制寄存器启用 SPI 模块，准备进行数据传输。

### 经典应用：QSPI Flash

QSPI Flash 是一种基于 Quad SPI 协议的闪存设备，广泛应用于嵌入式系统中。它通过增加数据线数量（IO0-IO3）来提升数据传输速率，适用于需要高速存储访问的应用场景，如图像处理、文件系统等。

这是 QSPI 最经典的应用场景之一，许多 MCU 都内置了 QSPI 控制器，支持直接访问外部 QSPI Flash 存储器，实现高速数据读写。相关的规范和实现细节可以参考 JEDEC 标准（如 JESD216）以及各个厂商的技术文档。在应用 QSPI Flash 之前，一般需要了解 QSPI **五阶段传输模型**：

```py
[指令 Opcode] → [地址 Address] → [交替字节 Alternate] → [空周期 Dummy] → [数据 Data]
```
- **线数表示法**：Cmd-Addr-Dummy-Data（如 1-1-4 表示指令1线、地址1线、Dummy 1线、数据4线）
- **Dummy Cycles**：Flash 内部准备数据所需的时钟周期数，频率越高/Dummy 越多，需通过寄存器或 SFDP 查询
- **Alternate Bytes**：常用于传输 Mode Bits，控制 Flash 进入连续读取（XIP）或切换工作模式

QSPI Flash 比较重要的一个特性是支持 **XIP（Execute In Place）**，允许 MCU 直接从 Flash 中执行代码，无需先将数据加载到 RAM 中。这对于存储大量代码或数据的应用非常有用，可以节省 RAM 资源并提升系统性能。


## 总结

不论怎么说，SPI 都不是一个特别困难的通信协议，复杂的是在这层“链路层”上跑的各种应用层协议（如 SD 卡协议、QSPI Flash 协议等）。但是，这些东西展开出来就非常复杂了，哪怕只是应用，也有大几十条的命令。所以，SPI 的核心其实就是一个非常简单的时钟同步的串行通信协议，真正的复杂性在于它的应用层协议和各种扩展模式（Dual/Quad SPI）。


## 参考文档
- [野火 SPI—读写串行FLASH](https://doc.embedfire.com/mcu/stm32/f103badao/std/zh/latest/book/SPI.html)
- [STM32F1 SPI 相关寄存器](https://www.st.com/resource/en/reference_manual/cd00171190.pdf)（RM0008）
- [jiegec Chen 的博客](https://jia.je/kb/hardware/spi.html?h=spi#sd)
