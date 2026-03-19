# EPUB 电子书阅读器（SF32-OED-EPD）

## 项目简介

本项目基于 [atomic14/diy-esp32-epub-reader](https://github.com/atomic14/diy-esp32-epub-reader) 进行适配，运行于 SiFli SF32-OED-EPD 系列开发板，用于 EPUB 电子书阅读与低功耗场景。

## 当前功能

### 电子书与文件系统

- 支持从内置文件系统与 TF 卡读取 EPUB（优先使用 TF 卡）。
- `disk/` 目录可打包示例 EPUB 到镜像中。

### 输入与显示

- 支持按键与触控双输入。
- 支持中文/英文显示。
- 为控制资源占用，默认未启用粗体/斜体中文字库。

### UI 页面与状态机

当前 UI 状态：

- `MAIN_PAGE`：主页面
- `SELECTING_EPUB`：书库页面
- `SELECTING_TABLE_CONTENTS`：目录页面
- `READING_EPUB`：阅读页面
- `SETTINGS_PAGE`：设置页面
- `WELCOME_PAGE`：欢迎页面
- `LOW_POWER_PAGE`：低电量页面
- `CHARGING_PAGE`：充电页面
- `SHUTDOWN_PAGE`：关机页面

默认开机进入 `MAIN_PAGE`。

### 主页面

主页面包含 3 个入口：

1. 打开书库
2. 继续阅读
3. 进入设置

说明：

- “继续阅读”基于本次运行期间最近一次成功进入阅读的书籍索引。
- 若当前无可用记录，则显示“无阅读记录”。

### 设置页面

支持以下设置项：

1. 触控开关（同步触控硬件上电/下电）
2. 超时策略：`5分钟 / 10分钟 / 30分钟 / 1小时 / 不关机`
3. 全刷周期：`5次 / 10次 / 20次 / 每次(0)`
4. 确认保存并返回主页面

### 阅读页覆盖操作层

在阅读页触发 `UPGLIDE`（上滑/映射动作）可呼出覆盖操作层，支持：

- 中心功能在“触控开关”与“全刷周期”间切换
- 跳页步进：`-5 / -1 / +1 / +5`
- 确认跳转到目标页
- 进入目录页
- 返回书库

### 书库页与目录页

- 书库页：每页 4 项，底部支持“上一页 / 主页面 / 下一页”。
- 目录页：每页 6 项，底部支持“上一页 / 书库 / 下一页”。
- 触控采用“先选中再确认”机制以降低误触。

### 电量与低功耗

- 页面顶部显示电量与充电状态。
- 低电量进入 `LOW_POWER_PAGE` 并抑制普通用户操作。
- 充电状态变化可触发页面刷新。
- 用户无操作达到设置超时后进入 `WELCOME_PAGE`。
- 主循环默认 5 小时无交互进入 `SHUTDOWN_PAGE`（常量 `TIMEOUT_SHUTDOWN_TIME`）。

---

## 使用指南

### 硬件连接

- 将开发板与墨水屏通过对应连接器连接，注意排线方向。
- `SF32-OED-EPD_V1.1` 与 `SF32-OED-EPD_V1.2` 可共用 `sf32-oed-epd_base` 软件板级配置。


### 编译与烧录

进入 `epdiy-epub/project` 后执行：

```bash
scons --board=sf32-oed-epd_v11 --board_search_path=.. -j8
```

编译完成后下载：

```bat
build_sf32-oed-epd_v11_hcpu\uart_download.bat
```

按脚本提示输入串口号。

参考：
[SiFli 官方编译/烧录文档](https://docs.sifli.com/projects/sdk/latest/sf32lb52x/quickstart/build.html)

### menuconfig

```bash
scons --board=sf32-oed-epd_v11 --board_search_path=.. --menuconfig
```


## 操作说明

### 按键动作语义

- `UP`：上移 / 上一项 / 向前翻页
- `DOWN`：下移 / 下一项 / 向后翻页
- `SELECT`：确认 / 进入
- `UPGLIDE`：阅读页呼出覆盖操作层

### 触控操作

- 主页面：点击左右区域切换选项，点击中间区域确认。
- 书库页：点击书籍项选中，再次点击确认打开。
- 目录页：点击目录项选中，再次点击确认跳转。
- 阅读页：左右区域翻页；上滑呼出覆盖层；覆盖层内执行跳页/目录/返回书库。

---

## 目录结构
```
│  
├──disk                             # 内置flash，存放少量EPUB电子书文件
├──lib                              # 存放项目依赖的第三方库或自定义基础库
│   ├──epdiy                        # EPD驱动核心库
│   ├──Epub                         # EPUB 解析专用库，负责解析 EPUB 格式文件（解压、读取目录、解析内容结构、提取文本 / 图片等），是阅读器功能基础。 
│   │   ├──EpubList                 # EPUB 文件管理模块 
│   │   ├──Renderer                 # 渲染器模块，将 EPUB 解析后的内容（文本、图片）转换为电子书可显示的格式
│   │   ├──RubbishHtmlParser        # HTML 解析模块，以 XHTML 格式存储，此模块负责解析 HTML 结构，提取文本、图片、样式
│   │   ├──ZipFile                  # 补充解压逻辑（配合 miniz），处理 EPUB 压缩包内的 HTML 及资源文件
│   ├──Fonts                        # 存放字体数据文件，储字符的像素映射
│   ├──Images                       # 图片处理模块，负责加载、解码
│   ├──miniz-2.2.0                  # 轻量级 ZIP 解压库，处理 EPUB 压缩包
│   ├──png                          # PNG 图像解码库，为 PNGHelper 提供底层解码能力。
│   └──tjpgd3                       # JPEG 图像解码库，为 JPEGHelper 提供底层解码能力。
|
├──project                          # 编译脚本为项目编译、调试、部署提供工具链支持
|
├──sf32-oed-epd_v11和sf32-oed-epd_v12                # `SF32-OED-EPD_V1.1`和`SF32-OED-EPD_V1.2`开发板公用的相关配置文件
├──sf32-oed-epd_v12_spi             # `SF32-OED-6'-EPD_V1.2`开发板搭配SPI墨水屏的相关配置文件
|
├──src                              # 项目核心源码目录，实现阅读器业务逻辑
    ├──boards                       # 硬件板级<br>
    │    ├──battery                 # 电池管理模块，包含电量检测
    │    ├──controls                # 输入控制模块（如按键、触摸），处理用户交互（翻页、选菜单等）
    │    └──display                 # 实现屏幕显示的最终输出，且可能针对不同屏幕型号做适配
    ├──epub_mem.c                   # 内存管理模块，为 EPUB 解析、渲染分配 / 释放内存，适配嵌入式设备内存限制。
    │       
    └──main.cpp                     # 程序入口（main 函数），初始化硬件、加载库、启动阅读器主逻辑（如打开 EPUB 文件、进入阅读界面 ）

```
---

## 二次开发

### 添加 EPUB

- 少量样书：放入 `disk/`，随文件系统镜像打包。
- 大量书籍：建议使用 TF 卡。

#### 添加新的屏幕屏驱

1. 复制已有屏驱配置文件并改名（建议以 `src/boards/display_dbi/epd_configs_opm060d.c` 为模板，按新屏型号命名）。
2. 根据屏幕波形文档，将波形数据转换为数组，例如：
   - 全刷波形：`static const uint8_t xxx_wave_forms_full[32][256] = {}`
   - 局刷波形：`static const uint8_t xxx_wave_forms_partial[12][256] = {}`
3. 按屏驱文档修改关键函数（波形选择、LUT 转换、时序频率、VCOM）。

 * 3.1多温区波形选择（可选)

    如果波形按温度分段，可组织为多组二维数组，通过温度选择对应波形：

```c
// 定义波形表条目结构体
typedef struct {
    int min_temp;
    int max_temp;
    uint32_t frame_count;
    const uint8_t (*wave_table)[256];
} WaveTableEntry;

// 原始波形数据
static const uint8_t te067xjhe_wave_full_0_5[45][256] = {};
static const uint8_t te067xjhe_wave_full_5_10[45][256] = {};
// ...
static const uint8_t te067xjhe_wave_full_50_100[45][256] = {};

// 全刷波形表 - 按温度区间组织
static const WaveTableEntry te067xjhe_wave_forms_full[] = {
    {0,   5,  45, &te067xjhe_wave_full_0_5[0]},
    {5,  10,  45, &te067xjhe_wave_full_5_10[0]},
    // ...
    {50, 100, 45, &te067xjhe_wave_full_50_100[0]},
};

static const uint8_t *p_current_wave_from = NULL;

uint32_t epd_wave_table_get_frames(int temperature, EpdDrawMode mode)
{
    const WaveTableEntry *wave_table;
    size_t table_size;

    wave_table = te067xjhe_wave_forms_full;
    table_size = sizeof(te067xjhe_wave_forms_full) / sizeof(WaveTableEntry);

    // 查找与温度匹配的区间
    for (size_t i = 0; i < table_size; i++) {
        if (temperature >= wave_table[i].min_temp && temperature < wave_table[i].max_temp) {
            p_current_wave_from = (const uint8_t *)(*wave_table[i].wave_table);
            return wave_table[i].frame_count;
        }
    }

    // 默认回退到第一组
    p_current_wave_from = (const uint8_t *)(*wave_table[0].wave_table);
    return wave_table[0].frame_count;
}
```

* 3.2波形转换为 32 位 Epic LUT

    将当前帧的 8 位波形转换为硬件使用的 32 位 LUT 值：

```c
void epd_wave_table_fill_lut(uint32_t *p_epic_lut, uint32_t frame_num)
{
    // 每帧波形长度为 256 字节
    const uint8_t *p_frame_wave = p_current_wave_from + (frame_num * 256);

    // Convert the 8-bit waveforms to 32-bit epic LUT values
    for (uint16_t i = 0; i < 256; i++)
        p_epic_lut[i] = p_frame_wave[i] << 3;
}
```

* 3.3调整时序与频率参数

    按屏驱规格书配置 `SDCLK -> sclk_freq`、`frame clock -> fclk_freq` 及相关时序参数：

```c
const EPD_TimingConfig *epd_get_timing_config(void)
{
    static const EPD_TimingConfig timing_config = {
        .sclk_freq = 24,  // 像素时钟（列时钟），单位 MHz
        .SDMODE = 0,      // SD 模式选择
        .LSL = 0,         // 行开始信号需要的 clock 数
        .LBL = 0,         // 行起始的空 clock 数
        .LDL = LCD_HOR_RES_MAX / 4, // 有效数据 clock 数（2bit+8数据线，需除以4）
        .LEL = 1,         // 行结束空 clock 数

        .fclk_freq = 83,  // 行时钟，单位 KHz
        .FSL = 1,         // 起始行信号需要的 clock 数
        .FBL = 3,         // 起始空行数量
        .FDL = LCD_VER_RES_MAX, // 有效数据行数
        .FEL = 5,         // 结束空行数量
    };

    return &timing_config;
}
```

* 3.4设置 VCOM 电压

    根据新屏规格书提供电子书显示所需参考电压：

```c
uint16_t epd_get_vcom_voltage(void)
{
    return 2100;
}
```

4. 在 `project/Kconfig.proj` 中新增屏幕宏与 `menuconfig` 选项，并将新配置文件加入编译条件。

#### 板子必须补充的 `Kconfig.proj` 配置
* 以`sf32-oed-epd_v12`板子为例
* 如果你新增了一个屏驱并希望在 `sf32-oed-epd_v12` 上可选可编译，`project/Kconfig.proj` 需要同步添加。建议按以下顺序处理：

1. 在 `choice "Custom LCD driver"` 下新增你的屏幕选项（建议命名带 `V12`，便于和 `V11` 区分）。
2. 在该选项中 `select` 对应触控驱动、EPD 类型宏和总线宏（一般为 `BSP_LCDC_USING_EPD_8BIT`）。
3. 在 `LCD_HOR_RES_MAX` / `LCD_VER_RES_MAX` / `LCD_DPI` 中补上新屏默认值。
4. 若希望 v12 默认选中新屏，在 `choice` 的 `default ... if BSP_USING_BOARD_SF32_OED_EPD_V12` 中切换为新宏。

示例（按你的新屏替换宏名与参数）：

```kconfig
choice
    prompt "Custom LCD driver"
    default LCD_USING_EPD_YOUR_PANEL_V12 if BSP_USING_BOARD_SF32_OED_EPD_V12

    config LCD_USING_EPD_YOUR_PANEL_V12
        bool "6.x rect electronic paper display(YourPanel 1234x567) for V1.2 board"
        select TSC_USING_GT967 if BSP_USING_TOUCHD
        select LCD_USING_OPM060D
        select BSP_LCDC_USING_EPD_8BIT
endchoice

config LCD_HOR_RES_MAX
    int
    default 1234 if LCD_USING_EPD_YOUR_PANEL_V12

config LCD_VER_RES_MAX
    int
    default 567 if LCD_USING_EPD_YOUR_PANEL_V12

config LCD_DPI
    int
    default 300 if LCD_USING_EPD_YOUR_PANEL_V12
```

完成后建议执行一次：

```bash
menuconfig --board=sf32-oed-epd_v12 --board_search_path=..
```

确认新屏选项可见并选中，然后再进行编译。

参考：
[SiFli 屏幕模块适配说明](https://wiki.sifli.com/tools/%E5%B1%8F%E5%B9%95%E8%B0%83%E8%AF%95/%E6%B7%BB%E5%8A%A0%E5%B1%8F%E5%B9%95%E6%A8%A1%E7%BB%84%EF%BC%883%20%E5%A4%96%E7%BD%AE%EF%BC%89.html)

### 3) 字体生成

可先生成 Unicode 区间，再由 TTF 生成字库头文件。

生成区间示例：

```bash
python3 get_intervals_from_font.py abc.ttf > interval.h
python3 generate_gb2312_L1_intervals.py
```

生成字体示例：

```bash
python3 fontconvert.py regular_font 15 abc.ttf > ../lib/Fonts/regular_font.h
```

注意：Unicode 区间必须按从小到大排序，否则字符检索会异常。

---

## 致谢

- 上游项目：[atomic14/diy-esp32-epub-reader](https://github.com/atomic14/diy-esp32-epub-reader)
- SiFli SDK 与开发文档支持
