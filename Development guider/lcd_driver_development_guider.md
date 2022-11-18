# ESP_LCD 驱动介绍
***

ESP 的 LCD 驱动位于 **ESP-IDF** 下的 [components/esp_lcd](https://github.com/espressif/esp-idf/tree/master/components/esp_lcd)，目前仅存在于 **release/v4.4 及以上**版本中。**esp_lcd** 能够驱动 ESP 系列芯片硬件所支持的 **I2C**、**SPI**、**8080** 以及 **RGB** 四种接口的 LCD 屏幕，各系列芯片所支持的 LCD 接口如下表所示。

|   SoC    |                          I2C 接口                           |                          SPI 接口                           |                          8080 接口                          |                          RGB 接口                           |
| -------- | ----------------------------------------------------------- | ----------------------------------------------------------- | ----------------------------------------------------------- | ----------------------------------------------------------- |
| ESP32    | ![supported](https://img.shields.io/badge/-Supported-green) | ![supported](https://img.shields.io/badge/-Supported-green) | ![supported](https://img.shields.io/badge/-Supported-green) |                                                             |
| ESP32-S2 | ![supported](https://img.shields.io/badge/-Supported-green) | ![supported](https://img.shields.io/badge/-Supported-green) | ![supported](https://img.shields.io/badge/-Supported-green) |                                                             |
| ESP32-S3 | ![supported](https://img.shields.io/badge/-Supported-green) | ![supported](https://img.shields.io/badge/-Supported-green) | ![supported](https://img.shields.io/badge/-Supported-green) | ![supported](https://img.shields.io/badge/-Supported-green) |
| ESP32-C3 | ![supported](https://img.shields.io/badge/-Supported-green) | ![supported](https://img.shields.io/badge/-Supported-green) |                                                             |                                                             |

各接口的 LCD 驱动应用示例参考 ESP-IDF 下的 [examples/peripherals/lcd](https://github.com/espressif/esp-idf/tree/master/examples/peripherals/lcd)，这些示例目前仅存在于 **release/v5.0** 及以上版本中，因为 **release/v4.4** 中 esp_lcd 的 API 名称与高版本基本一致，所以同样可以参考上述示例（两者的 API 实现上有一些区别），**后续均以 ESP-IDF release/5.0 为基础进行介绍**。

由于 RGB LCD 屏幕的驱动原理与其他接口屏幕有本质差异，下面将按照 **非 RGB 接口**和 **RGB 接口**分别进行介绍：

## 非 RGB 接口
***

### 硬件框架
<div align=center ><img src="./%E9%9D%9E%20RGB%20%E7%A1%AC%E4%BB%B6%E6%A1%86%E6%9E%B6.png" width=400/></div>

包含 I2C、SPI 以及 8080 接口，这类屏幕上的驱动 IC 内部使用一个整帧大小的显存 GRAM，ESP 只需要把**刷屏数据**（局部大小）传给驱动 IC，驱动 IC 会把数据保存到显存中，并按照自身的刷新速率把**显示数据**（整帧大小）显示到屏幕上。

### 软件流程

<div align=center ><img src="./%E9%9D%9E%20RGB%20%E8%BD%AF%E4%BB%B6%E6%B5%81%E7%A8%8B.png" width=150/></div>

1. **初始化总线**：对接口总线进行配置和初始化，若同一总线挂载多个设备则仅需初始化一次
2. **创建 esp_lcd_panel_io**：基于接口总线新建设备，生成 `esp_lcd_panel_io_handle_t` 类型变量并提供 `esp_lcd_panel_io_tx_param()` 和 `esp_lcd_panel_io_tx_color()` API 供后续过程使用
3. **配置 esp_lcd_panel**：通过 **esp_lcd_panel_io** 对 LCD 屏幕的寄存器进行配置，生成 `esp_lcd_panel_handle_t` 类型变量并提供 `esp_lcd_panel_draw_bitmap()` 等 API 实现刷屏等操作

## RGB 接口
***

### 硬件框架

<div align=center ><img src="./RGB%20%E7%A1%AC%E4%BB%B6%E6%A1%86%E6%9E%B6.png" width=400/></div>

这类屏幕上的驱动 IC 不使用显存 GRAM，ESP 在自身内部维护至少一个整帧大小的 GRAM （默认放置于 PSRAM 内），通过 RGB 接口每次将 GRAM 内全部的刷屏数据传给屏幕上的驱动 IC，驱动 IC 将其作为显示数据直接驱动显示电路工作。

### 软件流程

<div align=center ><img src="./RGB%20%E8%BD%AF%E4%BB%B6%E6%B5%81%E7%A8%8B.png" width=150/></div>

1. **配置 LCD 寄存器**（可选）：很多 RGB 接口的屏幕首先需要通过 SPI 接口对内部寄存器进行配置，这类屏幕的详细信息请参考[资料](https://focuslcds.com/3-wire-spi-parallel-rgb-interface-fan4213/)。由于该操作仅在 LCD 初始化时进行一次，推荐使用 IO（ESP 或 IO 扩展芯片）模拟 SPI 进行实现。
2. **创建 esp_lcd_panel**：对 ESP 的 RGB 接口的参数进行配置，创建 `esp_lcd_panel_handle_t` 类型变量并提供 `esp_lcd_panel_draw_bitmap()` 等 API 实现刷屏等操作

# SPI 接口代码详解
***

参考的示例程序位于 ESP-IDF 中 [examples/peripherals/lcd/spi_lcd_touch](https://github.com/espressif/esp-idf/tree/release/v5.0/examples/peripherals/lcd/spi_lcd_touch/main)，下面对代码中各阶段具体的配置参数进行讲解。

## SPI 总线初始化
***

如果其他设备和 LCD 挂在同一总线上，仅需对其初始化一次。

```
#include "driver/spi_master.h"          // 依赖的头文件

spi_bus_config_t bus_cfg = {
    .sclk_io_num = PIN_NUM_LCD_SCLK,    // 连接 LCD SCK（SCL） 信号的 IO 编号
    .mosi_io_num = PIN_NUM_LCD_MOSI,    // 连接 LCD MOSI（SDO） 信号的 IO 编号，
    .miso_io_num = PIN_NUM_LCD_MISO,    // 连接 LCD MISO（SDI） 信号的 IO 编号，如果不需要从 LCD 读取数据，可以设为 -1
    .quadwp_io_num = -1,                // 必须设置且为 -1
    .quadhd_io_num = -1,                // 必须设置且为 -1
    .max_transfer_sz = LCD_H_RES * 10 * sizeof(uint16_t),
                                        // 此参数表示单次刷屏允许的最大字节数，若后续采用 LVGL ，则通常设置为 LVGL buffer 大小
};
ESP_ERROR_CHECK(spi_bus_initialize(SPI2_HOST, &bus_cfg, SPI_DMA_CH_AUTO));
                                        // 第 1 个参数表示使用的 SPI host ID，后续创建 SPI master 设备同样会用到
                                        // 第 3 个参数表示使用的 DMA 通道号，默认设置为 SPI_DMA_CH_AUTO 即可
```

1. 如果 LCD 驱动 IC 配置为 **Serial Interface-I** 型（见 LCD 硬件详解），仅需设置 `mosi_io_num` 为其数据线 IO，将 `miso_io_num` 设置为 -1。

2. `max_transfer_sz` 参数仅用于驱动内部对用户传输数据的大小进行判断，若超出范围则报错，并不是内部创建 buffer 的大小，而且 SPI 只会在传输 PSRAM 内存时动态创建同等大小的 DMA buffer，因为 SPI 驱动不支持 DMA 传输 PSRAM 内存。**需注意**，单次刷屏的字节上限不仅受限于 `max_transfer_sz`，而且受限于硬件寄存器 `SPI_LL_DATA_MAX_BIT_LEN`（不同系列芯片数值不同，可在 ESP-IDF 中搜索到），**为保证程序正常运行**，它们的大小关系应满足 `单次传输字节数 <= max_transfer_sz <= 2^(SPI_LL_DATA_MAX_BIT_LEN - 3)`。

3. 目前 esp_lcd 不支持驱动 QSPI LCD，但是可以自行通过 SPI 实现驱动。

## 创建 panel_io
***

基于初始化好的 SPI host 创建一个 panel_io，每一个 **panel_io** 对应一个 **SPI 设备**。

```
#include "esp_lcd_panel_io.h"       // 依赖的头文件

static bool notify_lvgl_flush_ready(esp_lcd_panel_io_handle_t panel_io, esp_lcd_panel_io_event_data_t *edata, void *user_ctx)
{
    // 第 2 个参数无用，固定为 NULL
    // 第 3 个参数为 io_config 中传入的 user_ctx

    lv_disp_drv_t *disp_driver = (lv_disp_drv_t *)user_ctx;
    lv_disp_flush_ready(disp_driver);
    return false;                   // 无用，默认返回 false 即可
}

esp_lcd_panel_io_handle_t io_handle = NULL;
esp_lcd_panel_io_spi_config_t io_config = {
    .dc_gpio_num = PIN_NUM_LCD_DC,          // 连接 LCD DC（RS） 信号的 IO 编号，必须设置且 > -1
    .cs_gpio_num = PIN_NUM_LCD_CS,          // 连接 LCD CS 信号的 IO 编号
    .pclk_hz = LCD_PIXEL_CLOCK_HZ,          // SPI 的时钟频率，ESP 最高支持 80M（SPI_MASTER_FREQ_80M）
                                            // 需根据数据手册确定其最大值
    .lcd_cmd_bits = LCD_CMD_BITS,           // LCD 命令所需的二进制位数，应为 8 的整数倍，需根据数据手册确定
    .lcd_param_bits = LCD_PARAM_BITS,       // LCD 数据所需的二进制位数，应为 8 的整数倍，需根据数据手册确定
    .spi_mode = 0,                          // SPI 的模式，需根据数据手册确定，见后续详解
    .trans_queue_depth = 10,                // 此参数表示内部 SPI 设备的数据队列的深度，一般默认设为 10 即可
    .on_color_trans_done = notify_lvgl_flush_ready,
                                            // 此参数用于注册用户的回调函数，每次 SPI 传输完成就会调用
    .user_ctx = &disp_drv,                  // 此参数会被传入上面回调函数，作为第 3 个参数 user_ctx

    .flags = {                              // 以下均为 SPI 时序相关参数，0 表示否，1 表示是，需根据数据手册确定
        .cs_high_active = 0,                // CS 高电平使能，否则低电平使能
        .dc_low_on_data = 0,                // DC 低电平表示数据、高电平表示命令，否则高电平表示数据、低电平表示命令
        .lsb_first = 0,                     // 数据/命令传输时低位优先，否则高位优先
        .octal_mode = 0,                    // 8 线 SPI 模拟 8080 时序，一般不用设置
        .sio_mode = 0,                      // 通过一根数据线（MOSI）读写数据，对应于 Serial Interface I 型，否则为 II 型
    },
};
ESP_ERROR_CHECK(esp_lcd_new_panel_io_spi((esp_lcd_spi_bus_handle_t)LCD_HOST, &io_config, &io_handle));
                                            // 第 1 个参数为上一节初始化好的 SPI host ID
                                            // 第 3 个参数为创建好的 panel_io 设备
```

1. 目前 esp_lcd 不支持 SPI 三线（即 9 位模式，见 SPI LCD 硬件详解），因此 `lcd_cmd_bits` 和 `lcd_param_bits` 必须为 8 的整数倍（8、16、24）。

2. 基于 panel_io 可以使用以下两个 API 发送**命令**和**数据**：

    a. `esp_lcd_panel_io_tx_param()`：用于发送 LCD 配置相关的命令和参数，内部均调用 `spi_device_polling_transmit()` 实现数据传输，该函数只有数据传输完毕后才会返回

    b. `esp_lcd_panel_io_tx_color()`：用于发送 LCD 刷屏的命令和像素数据，通过 `spi_device_polling_transmit()` 发送命令，而通过 `spi_device_queue_trans()` 发送像素数据，该函数将像素缓存地址压入队列（深度由 `trans_queue_depth` 指定）成功后立刻返回。因此，**对于后续使用 LVGL 的程序**，必须在回调函数中调用 `lv_disp_flush_ready()`，并注册到上面配置代码的 `on_color_trans_done` 中。

3. **根据数据手册 SPI 时序确定配置参数**

    <div align=center ><img src="./st7789_spi_timing.png" width=600/></div>

    a. `spi_mode`: 取决于 SCK 时钟线的 CPOL（极性）和 CPHA（相位）。**CPOL** 可以简单理解为 CS 使能后 SCK 在第几个跳变沿采样，0 为第 1 个跳变沿，1 为第 2 个跳变沿；**CPHA** 可以简单理解为 SCK 空闲时电平，0 为低电平，1 为高电平。它们的对应关系如下表所示：

    | spi_mode | CPOL | CPHA |
    | :------: | :--: | :--: |
    | 0        | 0    | 0    |
    | 1        | 0    | 1    |
    | 2        | 1    | 0    |
    | 3        | 1    | 1    |

    上图是 ST7789 数据手册上部分 SPI 功能时序图，从图中可以看出，SCL 在 CS 使能后第 1 个跳变沿采样，所以 CPOL = 0， SCL 空闲时电平为 0，所以 CPHA 为 0，因此可以得到 `spi_mode = 0`（一般大部分屏幕都为 0）。

    b. `cs_high_active`: 从上图可以看出 CS 拉低时开始操作，因此 `cs_high_active = 0`（一般大部分屏幕都为 0）。

    c.