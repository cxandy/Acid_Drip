# 本地改动清单 / Local Modifications

> 记录 Acid Drip 固件在 git HEAD 基础上的所有改动 + 已定位的待修问题。
> 更新日期：2026-10-29 ｜ 编译验证：✅ **GitHub Actions 全绿**（见 §8）｜ FQBN `rp2040:rp2040:rpipico` ｜ 采样率 16384Hz
> 当前分支：`main`（领先上游 `origin/main` = `f88e64d` 10 个提交）

---

## 0. 状态速览

| 项目 | 状态 |
|---|---|
| 阶段 0 修复（跨核双渲染） | ✅ 代码已改，**CI 编译通过** |
| 硬件变体支持（GP8 / GP2 运行时切换） | ✅ 变体 A、B 均已实现并编译通过 |
| 文档一致性（README / 说明书） | ✅ 已对齐代码 |
| 酸音滤波器音档 | ✅ 已切回「旧引擎音档」（见 §3.4）|
| 编译验证 | ✅ **GitHub Actions 全部通过**（见 §8） |
| 固件产物 | ✅ CI 产出的两个 UF2（见 §8.2） |
| 与上游合并 | 零冲突已实测，但上游工作流会覆盖 PR（见 §9） |

---

## 1. 同步引脚：运行时选择（核心设计）

### 1.1 问题背景

原版固件里 **GP2 同时承担「鼓 + DRIFT 音频」和「同步时钟」两个职能**，靠板载 SPDT 开关分时复用。
上游的规避手段是 `if (!syncMode) bmStartAudio();`——**任何同步模式下鼓与 DRIFT 全哑**。

本 fork 的硬件改动是：**LCD RST 由 GP8 跳线到 3V3，腾出 GP8 做同步**，GP2 专归音频。

### 1.2 最终方案：单固件 + 开机菜单

不编译期宏，不刷两次。**同一个 UF2 同时兼容改装板和原版 PCB**，开机时用 pad 选。

| 变量 / 常量 | 位置 | 说明 |
|---|---|---|
| `uint8_t gSyncPin = SYNC_PIN_GP8;` | A:1434 / B:1283 | 运行时同步引脚，默认 GP8 |
| `#define SYNC_PIN_GP8 8` | A:491 / B:351 | 改装板 |
| `#define SYNC_PIN_GP2 2` | A:492 / B:352 | 原版 PCB |
| `PAD_SYNC_GP8_IN/OUT`、`PAD_SYNC_GP2_IN/OUT` | A:540-543 / B:397-400 | 开机菜单 4 个选项的 pad 编号（0/1/2/3）|

**开机手势**：按住 **pad 14（GP9）** 开机 → 青色 **SYNC SETUP** 菜单 → 按下列 pad 之一：

| Pad | `gSyncPin` | 方向 | 适用硬件 |
|---|---|---|---|
| pad 1 | GP8 | Sync In | 改装板 |
| pad 2 | GP8 | Sync Out | 改装板 |
| pad 3 | GP2 | Sync In | 原版 PCB（SPDT 拨到 SYNC 档）|
| pad 4 | GP2 | Sync Out | 原版 PCB（SPDT 拨到 SYNC 档）|

**什么都不按 = 鼓模式**，`gSyncPin` 保持默认 GP8，`syncMode = false`。

> 向后兼容：pad 1 / pad 2 的语义与原两选项菜单完全一致（= GP8），老用户无需改变习惯。

### 1.3 为什么 `TFT_RST` 保持 8

`TFT_RST` 在两个变体里都保留 **8**，不改 -1：

- **改装板**：LCD 的 RST 线已跳 3V3，驱动 GP8 做复位脉冲打不到面板，**无害**；
- **原版 PCB**：LCD 的 RST 线确实在 GP8，驱动它复位是**必需**的。

正因为同步引脚改成运行时变量，一份固件才能同时服务两块板。

### 1.4 错配的症状（重要）

菜单四个选项始终都显示，**软件无法自动识别板型**，选错会有明显症状：

| 硬件 | 误选 | 症状 | 正确选项 |
|---|---|---|---|
| 原版 PCB | pad 1 / pad 2（GP8）| GP8 是 LCD 复位脚，被配成 `INPUT_PULLDOWN` 后**屏幕全黑**；时钟也没接到任何插口 | pad 3 / pad 4 |
| 改装板 | pad 3 / pad 4（GP2）| GP2 没接同步插口，**时钟打到鼓/DRIFT 音频输出**；该模式下鼓静音 | pad 1 / pad 2 |

---

## 2. 阶段 0 修复：跨核双渲染（🔴 P0）

### 2.1 问题链条

fork 把 `bmStartAudio()` 改成**无条件调用** → `bmAlarmStarted = true` → `BeatMachine2.ino:2305`
的守卫 `if(!bmAlarmStarted) return;` **失效** → **core 1 开始渲染 drums + DRIFT 进 GP2**；
与此同时 core 0 的 `updateAudio()` 里 `if (!syncMode)` 的 fold 分支**也渲染一遍**折进 GP15。

原设计的 fold 分支注释明确写着它依赖 *"in sync mode bmStartAudio() is skipped"*——fork 把这个前提改掉了。

后果：两个核**同时推进同一份** DRIFT echo/verb 静态状态与鼓 voice 包络/滤波状态（数据竞争）；
鼓声翻倍且相位不一致；MIX EDIT trim 被应用两次；物理混合电位器在同步模式下再也关不掉鼓声。

### 2.2 修法：运行时条件分支（不是删除）

删除 fold 分支会**破坏原版 PCB 的同步行为**（那里 GP2 就是时钟脚，鼓必须折到 GP15 才听得见）。
所以改成按 `gSyncPin` 运行时判定：

```c
// A:6625
if (gSyncPin == SYNC_PIN_GP2 && syncMode) {
    // 原版 PCB：GP2 被时钟占用，鼓+DRIFT 折到 GP15
    ...
}
```

配套地，`bmStartAudio()` 改成有条件调用（**setup 与 doModeSwitch 两处都要改**）：

```c
// A:5097, A:6787 / B:4692, B:6338
if (gSyncPin != SYNC_PIN_GP2 || !syncMode) bmStartAudio();
```

于是形成两种**互斥且各自安全**的拓扑：

| `gSyncPin` | `syncMode` | 谁渲染鼓/DRIFT | 跨核安全性 |
|---|---|---|---|
| GP8 | true / false |  只有 core 1（→ GP2）| ✅ `bmStartAudio()` 执行，`bmAlarmStarted = true`，core 1 独占渲染 |
| GP2 | true |  只有 core 0（→ GP15，fold）| ✅ `bmStartAudio()` 被跳过，`bmAlarmStarted` 保持 false，core 1 的 `bmFillDrumBufferTo()` 直接 early-return |
| GP2 | false |  只有 core 1（→ GP2）| ✅ 同第一行 |

> 关键不变式：**`bmAlarmStarted` 为真 ⟺ 鼓由 core 1 渲染；为假 ⟺ 鼓由 core 0 的 fold 分支渲染。**
> 两条路径永不同步执行 → 没有数据竞争，也不会电平翻倍。

### 2.3 变体 B 的差异（未做，需 rebase 后处理）

变体 B 的 `updateAudio()` **没有 fold 分支**，所以：

- B 的 **GP8 模式**：鼓/DRIFT 照常走 GP2，有声 ✅（与 A 一致）
- B 的 **GP2 模式**：GP2 被时钟占用，**鼓静音**（= 原版行为），因为 B 里没有 fold 分支可用

要修需从 B 的 `BeatMachine2.ino` 抽取 `bmRenderDrumSample()` 到 core 0 可调的位置，
风险高且 B 反正要 rebase，**本轮不动**，已在代码注释与本文档中标注差异。

---

## 3. 阶段 0 其他改动

### 3.1 代码

| 改动 | 位置 | 说明 |
|---|---|---|
| 新增 `gSyncPin` 运行时变量 | A:1434 / B:1283 | 替换原来所有编译期 `SYNC_IN` / `SYNC_OUT` 宏 |
| 新增 `SYNC_PIN_GP8` / `SYNC_PIN_GP2` | A:491-492 / B:351-352 | |
| 新增 4 个 `PAD_SYNC_*` 常量 | A:540-543 / B:397-400 | 开机菜单选项 |
| 开机菜单扩为 4 项 SYNC SETUP | A:6720-6764 / B:6270-6316 | pad1/2 保持向后兼容 |
| 开机提示动态显示 GP8/GP2 | A:6761 / B:6313 | 原来写死 GP2 |
| `bmStartAudio()` 条件化 | A:5097, A:6787 / B:4692, B:6338 | 见 §2.2 |
| fold 分支改运行时条件 | A:6625 | 见 §2.2 |
| 重新引入 `bmRenderDrumSample` 声明 | A:238 | 带 `__not_in_flash_func`，**消除与 `BeatMachine2.ino:2313` 定义的属性不一致**（不一致会让 GCC 把函数放进 flash，core 0 音频热路径每采样都 XIP 停顿）|
| 删 `extern byte bmStepDiv;` | A:229 / B:141 | 全工程无定义（两变体都已清）|
| 删 `bool syncIn` | 主 sketch | 死变量，从未使用 |
| `BENCH_AUDIO_HEADROOM` 默认关闭 | A:148-159 | 原为 `1`（注释自称 "this isn't meant to ship"），改为 `#ifndef` 编译期开关，测量构建可 `-DBENCH_AUDIO_HEADROOM=1` 打开 |
| 头部注释 ACCENT EDIT→MIX EDIT | 主 sketch | 代码实为 MIX EDIT（`mixEditMode`），注释是 V4 遗留 |
| 头部接线注释对齐 | A:90-100 | 接线表 + SYNC SETUP 菜单说明 |
| 清理 sync 过期注释 | 主 sketch | 原描述旧 GP2 拓扑的约 40 行（含把开机手势误写为 "pads 14+13"——实际是**单独按 pad 14**）|
| Drift.ino 跨核归属注释 | Drift.ino | 原写"只有一个核调用"但主 sketch 有第二调用点，已随 fold 条件化补注说明 |
| Drift.ino "与鼓互斥"注释 | Drift.ino | 实际是 `mix += drift` 相加 |

### 3.2 文档（本轮同步）

| 文件 | 修正内容 |
|---|---|
| `README.md` | 顶部摘要 / 硬件表 / 接线块 / **整个 Sync 章节重写**：菜单名 `SYNC MODE`→`SYNC SETUP`、四选项表、双板对应关系、错配症状、PPQN 说明 |
| `说明书.md` | 顶部摘要 / 接线块 / 音频路径表 / **§三 开机与模式选择**（四选项表）/ **§八 同步模式详解**（重写）/ §10.2-10.4 排查 / §11.3 编码说明 / §十二 规格速查 |
| `LOCAL_MODIFICATIONS.md` | 本文件，全量重写 |

**本轮在文档里纠正的两处事实错误**：

1. **脉冲率**：原文写「24 PPQN，每四分音符 24 个脉冲，符合 TB-303 / 808 / 909 标准」——**错的**。
   实际 `SYNC_STEPS_PER_PULSE = 4`，而 `bpm2us(b) = 60000000/b/4`（16 分音符间隔），
   所以 **1 个门脉冲 = 4 个 16 分步进 = 1 拍 = 1 PPQN**。接入 24 PPQN 源会**跑快 12 倍**、BPM 卡在 300 上限。
   README 原文的「one gate is sent per sequencer step」同样不准（实际是每 4 步 1 个）。
2. **脉冲宽度**：原文写 2ms，代码里是 `SYNC_OUT_PULSE_MS = 15`（15ms）。

### 3.3 验证结果

| 检查 | 结果 |
|---|---|
| 大括号 / 圆括号 / 方括号平衡 | ✅ 变体 A 全部 12 个文件 0 差 |
| 删除符号的残留引用 | ✅ `bmStepDiv` / `syncIn` / `SYNC_IN` / `SYNC_OUT` 全清零 |
| 行尾完整性 | ✅ 所有 `.ino` / `.md` 保持原行尾（`.ino` 与 README/说明书 = CRLF 无 BOM；本文件 = LF）|
| 编码 | ✅ 两个 `.ino` 为干净 UTF-8（U+2014 分别 728 / 674 个，U+9225 乱码 0 个）；本轮文档编辑后 U+9225 仍为 0 |
| `digitalPinToInterrupt(pin)` 支持运行时变量 | ✅ arduino-pico 定义为 `(pin)`，已查 `cores/rp2040/Arduino.h` |
| `analogWriteFreq()` 对运行时 pin 安全 | ✅ arduino-pico 的 `analogWriteFreq` 写**全局** `analogFreq`，不按 pin 绑定（已读 `cores/rp2040/wiring_analog.cpp`）|
| `Adafruit_ILI9341` 支持 `_rst < 0` | ✅ 可退化为 `ILI9341_SWRESET` 软件复位（`Adafruit_ILI9341.cpp:194`）|
| **编译（变体 A）** | ✅ **通过**——GitHub Actions，见 §8 |
| **编译（变体 B）** | ✅ **通过**——GitHub Actions，见 §8 |

### 3.4 酸音滤波器：切回「旧引擎音档」（2026-10-30）

实测反馈：变体 A 开机播默认酸音**刺耳、难听**。查证结论——**不是同步/双渲染改动造成的**，
正常模式下酸音通路与上游**逐行一致**（上游自己的注释：*"NORMAL MODE: acid only, on the
exact proven 8-bit path (byte-identical to before DRIFT-in-sync existed)"*）。
真正原因是**换了引擎**，且默认编译下酸音走的是最新的 HP 滤波分支：

| | 旧 `Original_Sound` 变体（已删） | 变体 A 默认 | 变体 A 现在的音档 |
|---|---|---|---|
| 滤波 | 单级 2 极 SVF，12 dB/oct | 双级联 Q8 int32，24 dB/oct | 同左，但**无硬削波** |
| 峰值 | 自然限幅 | **±127 硬削波**（源码称 "scream flavour C"）| 级联软限幅 + `gResCap` 防自激 |
| 共振系数 | `effRes / 256` | `gAcidResCoeffQ8` 0.08–1.05（可自激）| `effRes / gResDiv`，`gResDiv` = 256 |
| 运行时可调 | 无 | SETTINGS 模式 | CUT / DECAY 两路可用，RES 那路空转 |

改动只有两个 define：

| 位置 | 改前 | 改后 | 作用 |
|---|---|---|---|
| `ACID_LPF_HP` | `1` | `0` | 落到 `ACID_LPF_4POLE` 分支：**去掉 ±127 硬削波**，峰值不再咬 |
| `ACID_LPF_RESDIV_SEED` | `200` | `256` | 共振系数 `effRes/gResDiv` 对齐旧引擎的 `effRes/256` |

`200 → 256` 这一步很关键：级联分支的系数是 `effRes/gResDiv`，种子 200 时默认 SAW 能到
`206/200 = 1.03`（>1 → 级自激尖啸），256 时封顶 `206/256 = 0.80`，与旧固件完全一致。
`gResDiv` **不存 EEPROM**（`FilterSettings` 只存 `envCut`/`decayDiv`/`cutMax`），种子一定生效。

这两个开关都是**上游作者自己留下的 A/B 通道**（源码注释：*"Set to 0 to fall back to the
8-bit cascade above for an instant A/B on hardware"*）。

> ⚠️ **两个副作用**：
> 1. **RES 旋钮方向反转**。HP 档下逆时针 = HP 共振最大（scream）；关掉 HP 后**顺时针 = 经典共振最大**。
> 2. **SETTINGS 模式（按 1+9）的 RES 那路变空转**——它驱动的 `gHpCutMax` 只喂 HP 分支。
>    CUT（包络扫频深度）和 DECAY（扫频速度）两路照常可用。
>
> 想切回上游默认音档：把 `ACID_LPF_HP` 改回 `1`、`ACID_LPF_RESDIV_SEED` 改回 `200`，
> CI 重编一版即可。

---

## 4. 同步时钟模型与已知限制

### 4.1 当前模型（DIRECT）

- **SYNC OUT**：`syncOutPulse()` 在每 4 个 16 分步进上拉高一次，`syncOutPoll()` 在
  `SYNC_OUT_PULSE_MS = 15` ms 后拉低——**真正的数字门脉冲**，与生态里其它合成器一致。
- **SYNC IN**：`syncPulseISR` 捕获上升沿，测量脉冲间隔并除以 `SYNC_STEPS_PER_PULSE`，
  **每来一个脉冲 = 1 拍**，内部再细分 4 个 16 分步进。

### 4.2 🔴 只支持 1 PPQN（上游也中招）

```
SYNC_STEPS_PER_PULSE = 4      // 假设外部每拍只发 1 个脉冲
bpm2us(b)            = 60000000 / b / 4   // 16 分音符间隔
```

喂标准 **24 PPQN DIN 同步**（TR-808 / 909 / TB-303）时：

```
120 BPM, 24 PPQN → 脉冲间隔 41667µs
seq.interval = 41667 / 4 = 10417µs   ← 实际每 16 分音符应为 125000µs
→ 音序器跑快 12 倍，BPM 读数被 clamp 到 300
```

这是上游 issue #5「外部时钟被忽略」的机制（**协议假设不匹配，不是引脚/电平问题**）。

**变体 B 已有 `syncDiv`**（`B:1927`、`B:5819-5829`）支持 1 / 2 / 24 PPQN，但无 UI 入口、需改源码重编译。
→ 计划：把 `syncDiv` 移植进变体 A 与上游，并加开机菜单项。

### 4.3 ⚠️ 变体 B 的 GP2 同步输出用硬件 PWM

B 的 `analogWrite(gSyncPin, duty)` + `analogWriteFreq()` 与 `PWMAudio bmPWM(2)` 可能抢同一个 PWM slice。
仅在 **GP2 模式**下出现——而该模式下 `bmStartAudio()` 被跳过，PWMAudio 未 `begin()`，理论上不冲突。
**未经硬件实测。**

---

## 5. 声音方案（二选一，各烧各的 UF2）

### A. 原版声音 — `Firmware/Acid_Drip_Drum_Acid_Drift_V5/`
= 上游最新代码（原版酸音引擎）+ 同步改动（运行时 `gSyncPin`、SYNC SETUP 四选项）+ MADCTL `0x40` 显示修复。
主 sketch 7280 行，另有 `BeatMachine2.ino`(2440) / `Drift.ino`(649) / `beats.h` / `sample0-7.h`。

### B. AcidBox 303 移植版 — `Firmware/Acid_Drip_V5_AcidBox/`
- 主 `.ino` 内嵌 AcidBox 303 引擎移植（synthVoice / rosic TeeBee 滤波器 / overdrive / 波形表）。
- **基于旧版上游分支**（`3738a4a` 那一代），保留 24 PPQN 硬件 PWM 同步输出。
- 相对当前上游**缺失** 6 块功能（见 §6.2），需 rebase 后才适合向上游贡献。
- 主 sketch 6648 行。

---

## 6. 已定位、尚未修复的问题

### 6.1 🔴 同步输入只支持 1 PPQN
见 §4.2。

### 6.2 🟡 变体 B 相对当前上游缺失的功能

| 缺失 | 符号 | 上游引用数 |
|---|---|---:|
| SETTINGS 模式 | `setEditMode` | 19 |
| 共振分频 | `gResDiv` | 12 |
| 包络归一化 | `gEnvCutNorm` | 10 |
| Q8 共振系数 | `gAcidResCoeffQ8` | 6 |
| 4 极级联 / 高精度 HP | `ACID_LPF_4POLE` / `ACID_LPF_HP` | 3 / 3 |
| ISR 时间戳同步输入 | `gSyncPulseMicros` | 4 |

变体 B 自身待修：`B:6012` 每采样调 `expf()`；`B:4947` RES 负半程在 303 音色上无效；
`g303AmpLP` 仅在换音色时复位导致同音色连发 amp env 不从 0 重启。

### 6.3 🟡 变体 B 的 GP2 模式没有 fold 分支
见 §2.3。

### 6.4 🟡 其他

- 上游 issue #8「FX 切换爆音」未处理。
- 上游**无 LICENSE 文件**（GitHub `license: null`），公开再分发前需确认授权。
- 仓库跟踪了 39 个 KiCad 备份 zip（`.gitignore` 后加，管不住已提交文件）。
- `Acid_Drip_PCB- Kicad/Acid_Synth/Acid_Synth.zip`（222KB）已从 git 删除但仍留在磁盘上。
- `Acid_Drip_Quick_Reference_Guide_GP8.pdf` 是只讲 GP8 的旧速查卡，**未包含四选项菜单**，本轮未重新生成（缺 PDF 生成工具链）。
  上游原版速查卡 `Acid_Drip_Quick_Reference_Guide.pdf` 未改动。

---

## 7. 音频引脚分配

| 信号 | 引脚 | 说明 |
|---|---|---|
| ACID 音频 | **GP15** | Mozzi `MOZZI_AUDIO_PIN_1 15`，100R + 10nF RC 后到 3.5mm |
| 鼓 + DRIFT 音频 | **GP2** | `PWMAudio bmPWM(2)`；DRIFT 加性混入鼓缓冲，470R 后到 3.5mm |
| 同步 IN/OUT | **GP8 或 GP2** | **运行时选择**（见 §1.2）|
| TFT RST | GP8（改装板已跳 3V3）| 两个变体的 `TFT_RST` 都保持 8 |
| 混合 | 物理电位器 | PCB 上混合 GP15 vs GP2；电位器拧到 ACID 端会**短路 GP2** |

---

## 8. 构建产物与 CI

### 8.1 GitHub Actions（当前唯一验证过的编译途径）

本机没装 arduino-cli / rp2040 core，编译统一交给 GitHub Actions：
`.github/workflows/build-firmware.yml`（push 到 main、任何 PR、手动触发都会跑）。

它做四件事：

1. 装 `arduino-cli` 1.1.1；
2. 加 Philhower 的额外索引 URL，装核心 **`rp2040:rp2040`**；
3. 装库：`Mozzi` 2.0.4（自动带 `FixMath`）、`Adafruit GFX Library`、`Adafruit ILI9341`；
4. 编译两个变体，把 UF2 作为 artifact 上传（artifact 名 `acid-drip-firmware`）。

**踩过并已修好的三个坑**：

- RP2040 核心**不在 Arduino 默认索引里**，要用额外索引
  `https://github.com/earlephilhower/arduino-pico/releases/download/global/package_rp2040_index.json`；
  该索引里的包名是 **`rp2040`** 而不是 `arduino`，所以核心 id 是 `rp2040:rp2040`，
  FQBN 是 `rp2040:rp2040:rpipico`（`arduino:rp2040` 会报 "Platform not found"）。
- `arduino/setup-arduino-cli@v2` 的版本输入名是 **`version`**，不是 `arduino-cli-version`。
- **必须用 `--build-path`，不能用 `--build-property build.path=...`**。
  后者只改了链接步骤看到的路径，而核心归档仍写在 arduino-cli 自己的内部目录里，
  链接会报 `ld: cannot find <build.path>/core/core.a`。

### 8.2 编译结果

最新：run `40972460342`（旧引擎音档，全绿）。

| 变体 | Flash | RAM | 产物 |
|---|---|---|---|
| A 原版声音 | 152772 B（7%）| 57468 B（21%）| `Acid_Drip_V5_Original_Sound.uf2` |
| B AcidBox 303 | 153172 B（7%）| 70048 B（26%）| `Acid_Drip_V5_AcidBox_303.uf2` |

> **UF2 不是逐字节可复现的**：`ar` 归档会内嵌成员时间戳，同一份源码两次构建的哈希也不同。
> 所以判断「变体改了没有」要看源码 diff，不要看 UF2 哈希。变体 A 的音档切换确实生效，
> 已用 `git diff` 核对（两次构建之间只有变体 A 的 `.ino` 变化）+ UF2 哈希变化双重确认。

除 Mozzi 库自身的 `#warning`（RP2040 平台未实现自动随机种子）外**无任何警告**。
变体 B 的 sketch 文件夹名与主 .ino 名不一致，CI 里先复制成同名目录再编译。

### 8.3 仓库里的旧产物（已过时，别烧）

| 产物 | 时间 | 实际对应 |
|---|---|---|
| `Firmware/build_out/Acid_Drip_V5_Drum_Acid_Drift.ino.uf2` | 2026-09-28 18:17:38 | **变体 B** |
| `Firmware/build_out_orig/Acid_Drip_V5_Original_Sound.ino.uf2` | 2026-09-28 18:17:12 | **已删除的中间版本** |

变体 A 在这棵树里**从未被编译过**——两个 map 都缺失变体 A 独有的全部符号
（`syncOutPulseOffMs`、`gResDiv`、`bench0_avgUs`、`bmRenderDrumSample`、
`gEnvCutNorm`、`gSyncPulseMicros` 全部 0 命中）。

→ **烧录请用 CI artifact**，仓库里这两个旧 UF2 都不能代表当前代码。

> `.gitignore` 把 `*.uf2` / `*.bin` / `*.elf` / `*.map` 全部排除，固件不进版本库。
> 需要 UF2 时从 Actions artifact 下载（本地副本在 `Firmware/build_out/ci/`，该目录同样被忽略）。

### 8.4 本地编译命令（需先装 `arduino-cli` + `rp2040:rp2040` core）

```
arduino-cli core update-index \
  --additional-urls https://github.com/earlephilhower/arduino-pico/releases/download/global/package_rp2040_index.json
arduino-cli core install rp2040:rp2040
arduino-cli lib install "Mozzi" "Adafruit GFX Library" "Adafruit ILI9341"

arduino-cli compile --fqbn rp2040:rp2040:rpipico \
  --build-path ../build_out \
  Firmware/Acid_Drip_Drum_Acid_Drift_V5
```

---

## 9. 与上游的关系

- 上游：`lonesoulsurfer/Acid_Drip_Bassline_and_Drum_Synth`（120 stars，最后推送 2026-09-13）
- 本地 `main` 领先上游 5 个提交；`git merge origin/main` **实测零冲突**（git 正确识别
  `src/` → `Firmware/` 重命名，无重复副本）。
- ⚠️ 上游工作流是 **「Delete 目录 + Add files via upload」**（96 个提交全是作者一人），
  任何针对 `src/` 内文件的 PR 都会在他下次整体上传时被覆盖。
  → 对策见《与上游合并规划》：只提小而聚焦的 bug 修复 PR，不提目录改名。

---

## 10. 文件清单

```
Firmware/Acid_Drip_Drum_Acid_Drift_V5/     # 变体 A（原声 + 运行时 gSyncPin + 阶段 0 修复）
Firmware/Acid_Drip_V5_AcidBox/             # 变体 B（AcidBox 303，待 rebase）
Firmware/Acid_Drip_Drum_Acid_V4/           # 上游 V4（未改动）
.github/workflows/build-firmware.yml        # CI：编译两个变体并上传 UF2 artifact
Firmware/build_out/ci/                     # 本地 CI 产物副本（gitignored，不入库）
README.md                                   # 已更新
Acid_Drip_Quick_Reference_Guide_GP8.pdf     # GP8 速查卡（**旧版，无四选项菜单**）
LOCAL_MODIFICATIONS.md                       # 本文件
说明书.md                                    # 中文用户手册（已更新）
```

> **编码说明**：两个 `.ino` 均为干净 UTF-8（实测 U+2014 分别 713 / 661 个，
> U+9225 乱码 **0** 个）。此前"原版声音 .ino 含字面乱码 `鈥?`"的说法是
> **PowerShell 控制台按非 UTF-8 解码造成的显示假象**，文件本身没问题。
> 字面 U+9225 目前只存在于本文件里对它的引用。
