# ESPHome 86Box 空调面板

这是一个给 ESP32-S3 86Box 开发板使用的 ESPHome 项目，目标硬件是 480x480 ST7701S RGB 屏 + FT6336U 触摸。当前界面是一个 Home Assistant 空调控制面板，控制实体为：

```yaml
climate.lemesh_cn_2021901176_air02
```

参考工程是 `ESP32S3_3.95In_Box_rev2/`。ESPHome 项目没有复制完整 IDF 示例代码，而是尽量使用 ESPHome 内置的 `st7701s`、`ft63x6` 和 `lvgl` 组件。

## 文件

- `esp32s3-st7701s-ft6336u.yaml`：ESPHome 主配置。
- `ESP32S3_3.95In_Box_rev2/`：原厂/示例 IDF 工程，用于核对引脚、屏幕初始化、背光和触摸配置。
- `template.html`：早期 UI 参考稿。

## 硬件配置

主控：

- ESP32-S3
- 16MB Flash
- Octal PSRAM 80MHz
- ESP-IDF framework
- CPU 240MHz

屏幕：

- ST7701S RGB LCD
- 分辨率 `480x480`
- 背光 `GPIO13`
- SPI 初始化总线：`CLK GPIO38`、`MOSI GPIO39`、`CS GPIO45`
- RGB 控制脚：
  - DE `GPIO47`
  - PCLK `GPIO48`
  - HSYNC `GPIO14`
  - VSYNC `GPIO21`
- RGB data pins：
  - `GPIO0, GPIO12, GPIO11, GPIO10, GPIO9, GPIO46, GPIO3, GPIO20`
  - `GPIO19, GPIO8, GPIO18, GPIO17, GPIO16, GPIO15, GPIO7, GPIO6`

触摸：

- FT6336U / FT63X6
- I2C：`SDA GPIO5`、`SCL GPIO4`
- 地址：`0x38`

## UI 功能

当前 LVGL 界面包含：

- 当前模式状态：`MODE COOL/HEAT/FAN/DRY/OFF`
- 目标温度大号显示
- 当前温度显示
- `-` / 电源 / `+` 三个温控按钮
- 底部四个模式按钮：
  - `COOL`
  - `HEAT`
  - `FAN`
  - `DRY`

温度调整每次变化 1 度，并调用 Home Assistant 的 `climate.set_temperature`。

电源按钮逻辑：

- 当前开机时点击，发送 `hvac_mode: off`
- 当前关机时点击，发送 `hvac_mode: cool`

模式按钮逻辑：

- 点击模式按钮会立即更新本地界面高亮，并调用 `climate.set_hvac_mode`
- Home Assistant 状态同步回来后，也会调用同一套高亮脚本
- 只有当前模式按钮高亮，其他模式按钮保持灰色

模式高亮由这些脚本统一维护：

- `show_mode_none`
- `show_mode_cool`
- `show_mode_heat`
- `show_mode_fan`
- `show_mode_dry`

这样可以避免“按钮点过就一直亮”的问题，最终显示以实际 HA 空调状态为准。

## 休眠

LVGL 配置了 20 秒无操作后休眠：

- `on_idle: 20s`
- 触发 `lvgl.pause`
- pause 时关闭 `LCD Backlight`
- 点击屏幕后通过 `resume_on_input` 唤醒
- resume 时重新打开背光

第一次触摸主要用于唤醒屏幕，不应误触发底层空调按钮。

## 旋转和颜色

当前配置面向较新的 ESPHome：

```yaml
lvgl:
  rotation: 180
```

新版 ESPHome 不允许在 `display:` 中设置 `rotation` 后再交给 LVGL 使用，因此旋转放在 `lvgl:` 下。

颜色相关关键点：

- `display.color_order: RGB`
- `lvgl.byte_order: little_endian`

之前颜色异常的原因是 RGB565 字节序不匹配。`byte_order: little_endian` 会让生成的 LVGL 配置使用 `LV_COLOR_16_SWAP 0`，这块屏幕下颜色才正常。

## 编译

本机 ESPHome 工具位于：

```powershell
C:\Users\henry\Documents\Python\my_env\Scripts\esphome.exe
```

常用命令：

```powershell
& 'C:\Users\henry\Documents\Python\my_env\Scripts\esphome.exe' config esp32s3-st7701s-ft6336u.yaml
& 'C:\Users\henry\Documents\Python\my_env\Scripts\esphome.exe' compile esp32s3-st7701s-ft6336u.yaml
& 'C:\Users\henry\Documents\Python\my_env\Scripts\esphome.exe' upload esp32s3-st7701s-ft6336u.yaml --device COM3
& 'C:\Users\henry\Documents\Python\my_env\Scripts\esphome.exe' logs esp32s3-st7701s-ft6336u.yaml --device COM3
```

注意：本地虚拟环境曾经是 ESPHome `2025.8.2`，该版本可能不支持 `lvgl.rotation`。如果本地报：

```text
[rotation] is an invalid option for [lvgl]
```

说明本地 ESPHome 版本偏旧。Home Assistant 里的新版 ESPHome 要求把旋转放在 `lvgl:`，也就是当前写法。

## Home Assistant 依赖

需要 ESPHome API 正常连入 HA。配置中使用了 Home Assistant sensor/text_sensor 读取：

- `current_temperature`
- `temperature`
- climate 当前状态，例如 `cool`、`heat`、`fan_only`、`dry`、`off`

ESPHome 设备必须在 Home Assistant 中正常添加，否则屏幕只能显示初始值，无法同步空调状态。

## 已知注意事项

- `GPIO45` 是 strapping pin，ESPHome 会警告，当前沿用示例工程用于 LCD CS。
- 背光是二值开关，不是 PWM 调光。
- 中文字体曾出现方块，当前 UI 使用英文文本和 Montserrat 字体。
- 示例工程使用 RGB panel + ST7701 初始化表；当前 YAML 的 `init_sequence` 是从示例工程翻译而来，并保留 `COLMOD 0x60`。
- 如果升级 ESPHome 后 LVGL schema 继续变化，优先以 HA 中最新版 ESPHome 的报错为准。

