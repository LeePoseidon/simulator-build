# simulator-build

给 [Eden](https://git.eden-emu.dev/eden-emu/eden)（Switch 模拟器）打补丁并用 GitHub Actions 构建 Windows 版。

## 目录

```
patches/
  0001-sdl2-switch-imu-calibration-fallback.patch   # SDL2 版：IMU 标定兜底（对应老版本 Eden / v0.2.1 及更早）
  0001-sdl3-switch-imu-calibration-fallback.patch   # SDL3 版：同上（对应 master / 夜版）
  0002-eden-motion-and-msvc-fixes.patch             # Eden 本体：编译修复 + 体感三合一（详见文件头注释）
.github/workflows/
  build-nightly-clangcl.yml                         # 夜版（master + SDL3），clang-cl，主力路线
  build-nightly-msvc.yml                            # 夜版（master + SDL3），纯 MSVC
  build-windows.yml                                 # 老版本 / SDL2 路线
```

## 根因

SDL 的 Switch 驱动在 `LoadIMUCalibration()` 里用 SPI 标定区算缩放系数：

```c
fGyroScaleX = MULT / (SensCoeffX - RawX) * PI / 180;
```

正品 Pro Controller 的标定区有值；**克隆手柄（北通等伪装 `057E:2009`）的标定区读回全 0**，
除数变成 0 → 系数 `inf`/`NaN` → 传感器数据全部 `NaN` → Eden 里体感恒定无输入，而按键、摇杆走的是
另一条通路，完全不受影响。补丁在除数不可信时回退到 SDL 自己 `else` 分支用的出厂默认系数。

## 用法

1. Actions → 左侧 `Build Eden Nightly (clang-cl)` → **Run workflow**
   - `eden_ref` 留空 = master；也可以填 tag（如 `v0.2.1`，但要换成 SDL2 的补丁）
   - `sdl_version` 留空 = 3.4.14
2. 跑完在 Artifacts 下载 `eden-nightly-clangcl-beitong-fix`。
3. 进游戏后确认体感：Eden 日志里应有 `Controller motion capabilities: accel=1 gyro=1`。

## 回弹（大角度运动时明显）—— 已定位并修复

**结论：北通这颗 IMU 报出的角速度约是真实值的 2 倍**（量程档位是任天堂那颗的一半，同样的角速度
产生两倍原始计数值，而我们套的是任天堂的换算系数）。Eden 用陀螺仪积分姿态、用加速度计测重力方向
做漂移修正，积分角度一旦是真实的 2 倍，修正项就会一直往回拽 —— 表现为「大角度转动后回弹」。

这也解释了为什么**只有上下（俯仰）回弹、左右（偏航）不回**：偏航是绕重力轴转，加速度计物理上看不见，
也就没法纠；俯仰能看见，所以拽得回来。

实测数据（用加速度计当尺子，量陀螺仪积分）：转动 65° 被报成 130°，比值 2.06 / 2.00 / 2.12。
把增益设成 0.5 后同法复测，比值变成 1.01 / 1.00，回弹消失。

`0002` 合并补丁已把默认增益定为 0.5，**直接用 eden.exe 即可，不需要任何环境变量**。
只有换用手柄或想复测时才用下面的钩子（运行时读取，改完重启程序生效）：

| 变量 | 默认 | 作用 |
|---|---|---|
| `EDEN_MOTION_GYRO_GAIN` | `0.5` | 陀螺仪乘法增益；设 `1.0` 回到未补偿的原始值 |
| `EDEN_MOTION_LOG` | 关 | 设 `1` 后每秒打一行 `[motion] ... turned=(x,y,z)deg` |
| `EDEN_MOTION_KP` | `0.3` | 加速度计回正强度，`0` = 纯陀螺仪积分（会缓慢漂移） |

### 换手柄/复测时怎么重新标定

1. `$env:EDEN_MOTION_LOG="1"; $env:EDEN_MOTION_GYRO_GAIN="1"; .\eden.exe`
2. `turned` 是从启动起的累计转角（度，看两次之间的差值）。绕一个轴**精确转 90°**，对比增量。
3. `新增益 = 90 / 实测增量`，例如实测 178° → `90/178 ≈ 0.5`。

正常手柄（任天堂正品）无需补偿，用 `EDEN_MOTION_GYRO_GAIN=1` 即可。

## 注意

- **不要用 `YUZU_USE_BUNDLED_SDL3=ON`**：那条路会拉 Eden 预编译的 SDL 包（`sdl3-ci`），补丁打不进去。
  必须 `YUZU_USE_BUNDLED_SDL3=OFF` + `SDL3_CUSTOM_DIR` 指向自己克隆并打好补丁的 SDL 源码树。
- 工作流已加 `-DSDL3_FORCE_BUNDLED=ON`。原因：`AddPackage` 里先 `find_package(SDL3 3.2.10)` 再判断
  `SDL3_CUSTOM_DIR`，runner 上若存在系统 SDL3 就会静默走系统库、补丁被忽略，而构建仍显示成功。
  configure 后会断言 `SDL3_CUSTOM_DIR` 指向 `sdl3/` 且 `SDL3_FORCE_BUNDLED:BOOL=ON`。
- 补丁文件必须是 **LF** 换行、且针对正确版本；版本对不上时工作流会先试 `--ignore-whitespace` 兜底，
  再失败就报错停住，不会带着未打补丁的 SDL 硬编下去。
- 这个补丁只覆盖「标定区读回全 0」这一种失败形态。若 SPI 读请求本身失败，SDL 自己会走进默认值分支，
  本来就没问题；若你的手柄体感仍然不动，先看日志里 `accel=` / `gyro=` 是不是 0。
