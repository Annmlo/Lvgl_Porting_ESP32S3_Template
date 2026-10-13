# LVGL Porting ESP32-S3 Template

微雪 **ESP32-S3-Touch-LCD-4.3**（800×480 RGB 屏 + GT911 触摸）的 LVGL 移植起手模板。

基于微雪官方 `08_lvgl_Porting` 例程整理，**`sdkconfig` 已调好并一并提交**，
clone 下来直接 `idf.py build flash monitor` 就能点亮屏幕，不需要再翻 menuconfig。

---

## 硬件

| 项目 | 参数 |
|---|---|
| 主控 | ESP32-S3（Octal PSRAM 80 MHz，Flash 8 MB） |
| 屏幕 | 4.3 寸 RGB LCD，800×480 横屏，RGB565，ST7701 驱动 |
| 触摸 | GT911（I2C） |
| 接口 | USB 串口（CH343 / CH340，视板子批次而定） |

## 软件

| 项目 | 版本 |
|---|---|
| ESP-IDF | **5.5.0**（`main/idf_component.yml` 要求 ≥ 5.1.0） |
| LVGL | **8.4.0**（已内置于 `components/lvgl__lvgl/`，不联网拉取） |
| esp_lcd_touch | 1.1.2 |
| esp_lcd_touch_gt911 | 1.1.1~1 |

---

## 目录结构

```
.
├── main/                              # ★ 应用代码，改这里
│   ├── main.c                         # 入口：初始化 LVGL、跑 lv_timer_handler 任务
│   ├── lvgl_port.c/.h                 # LVGL 移植层（tick / 互斥锁 / 显示与触摸注册）
│   ├── waveshare_rgb_lcd_port.c/.h    # RGB LCD + GT911 驱动层
│   ├── Kconfig.projbuild              # menuconfig 里的 "Example Configuration"
│   ├── CMakeLists.txt
│   └── idf_component.yml
├── components/                        # 本地内置组件（type: local，不走 registry）
│   ├── lvgl__lvgl/                    # LVGL 8.4.0 全量源码
│   ├── espressif__esp_lcd_touch/
│   └── espressif__esp_lcd_touch_gt911/
├── sdkconfig                          # ★ 已调好的板级配置（务必保留）
├── sdkconfig.defaults                 # 配置模板
├── dependencies.lock                  # 组件依赖锁定
├── CMakeLists.txt
├── README_upstream_08_lvgl_Porting.md # 上游例程原始说明（Espressif RGB Avoid Tearing Example）
└── .gitignore
```

---

## 快速开始

```bash
# 1. 激活 ESP-IDF 5.5 环境（Windows PowerShell）
#    注意：必须用 PowerShell，不要在 Git Bash 里跑 idf.py（MSYSTEM 会导致 idf_tools.py 直接 fatal）

# 2. 编译
idf.py build

# 3. 烧录 + 串口监视
idf.py -p COMxx flash monitor

# 退出 monitor：Ctrl + ]
```

## 关键配置（已包含在 sdkconfig 中）

| 配置项 | 值 | 说明 |
|---|---|---|
| `CONFIG_EXAMPLE_LVGL_PORT_AVOID_TEAR_MODE` | **3** | 免撕裂模式 3 = `direct_mode`，RGB 屏必选，否则撕裂 |
| `CONFIG_EXAMPLE_LVGL_PORT_ROTATION_DEGREE` | 0 | 横屏，不旋转 |
| `CONFIG_EXAMPLE_LCD_RGB_BOUNCE_BUFFER_HEIGHT` | 10 | bounce buffer 行数；**屏闪明显时可加大**（占内部 SRAM） |
| `CONFIG_LV_COLOR_DEPTH` | 16 | RGB565 |
| `CONFIG_SPIRAM_MODE_OCT` + `CONFIG_SPIRAM_SPEED_80M` | y | Octal PSRAM 80 MHz，帧缓冲放这里 |
| `CONFIG_LV_USE_METER` | y | 表盘控件（仪表盘 UI 常用） |
| `CONFIG_LV_USE_PERF_MONITOR` | y | 右上角/右下角帧率与 CPU 占用浮层 |

> 想改配置：`idf.py menuconfig` → `Example Configuration`（由 `main/Kconfig.projbuild` 定义）。
> 改完记得把 `sdkconfig` 一起提交，否则别人 clone 下来配置就丢了。

---

## 当作模板用：移植 PC 模拟器上调好的界面

1. 把新 UI 的 `*.c / *.h` 放进 `main/`；
2. 在 `main/CMakeLists.txt` 的 `SRCS` 里加上新文件；
3. 在 `main.c` 里调用你的创建函数，**必须包在锁里**：

```c
lvgl_port_lock(-1);
my_ui_create();
lvgl_port_unlock();
```

4. 触摸/按键回调里凡是碰 `lv_...` 的，也要用同一把锁。

### 版本落差提醒（PC 模拟器多为 LVGL 10）

| LVGL 10 | LVGL 8.4 |
|---|---|
| `lv_screen_active()` | `lv_scr_act()` |
| `lv_indev_active()` | `lv_indev_get_act()` |
| `lv_obj_set_scrollable(o, false)` | `lv_obj_clear_flag(o, LV_OBJ_FLAG_SCROLLABLE)` |
| `lv_anim_set_duration()` | `lv_anim_set_time()` |
| `lv_anim_set_reverse_duration()` | `lv_anim_set_playback_time()` |
| `lv_obj_set_style_transform_scale_x/y()` | `lv_obj_set_style_transform_zoom()`（256 = 100%） |
| `lv_scale_*` | `lv_meter_*`（8.4 没有 `lv_scale`） |
| `lv_label_set_text_fmt(..., "%f")` | 改成整数格式化（8.4 默认不带浮点 printf） |

完整映射表、以及 LVGL 8.4 上**最容易被忽略的四个性能陷阱**
（`lv_obj_set_style_*` 无条件全对象失效、`lv_label` 的 `ext_draw_size`、
失效区域自动外扩 5 px、`flush_cb` 是**按区域**而不是按帧调用），
见配套 skill **`lvgl-port-84`** 的 `references/` 目录。

---

## 已知注意事项

- **`dependencies.lock` 里记录了生成时的本机绝对路径**（上游微雪例程就是这样）。
  它只是来源信息，不影响编译；首次 `idf.py build` 会自动更新，也可以直接删掉重建。
  组件都在 `components/` 里本地提供（`type: local`），**编译不需要联网**。
- **`sdkconfig` 是刻意提交的**，别把它加进 `.gitignore`。
- `.vscode/` 未提交（里面是各人本机的 IDF / 调试器绝对路径），`.clangd` 已提交。

## 来源与许可

| 部分 | 来源 | 许可 |
|---|---|---|
| LVGL 8.4.0 | https://github.com/lvgl/lvgl | MIT |
| esp_lcd_touch / esp_lcd_touch_gt911 | Espressif Component Registry | Apache-2.0 |
| `main/` 移植层与例程 | 微雪电子 `ESP32-S3-Touch-LCD-4.3-Demo` → `ESP-IDF/08_lvgl_Porting` | 见微雪官方仓库说明 |

上游例程原始说明保留在 [`README_upstream_08_lvgl_Porting.md`](README_upstream_08_lvgl_Porting.md)。
