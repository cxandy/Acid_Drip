# 本地改动清单 / Local Modifications

> 记录 Acid Drip V5 自 git HEAD（`3738a4a`，原始上传版）以来的所有固件 + 文档改动。
> 日期：2026-09-28 ｜ 编译器：arduino-cli，FQBN `rp2040:rp2040:rpipico` ｜ 采样率 16384Hz

---

## 1. 同步引脚改动（两个声音方案工程都已改）

根因：原版固件中 GP2 同时承担 **鼓+DRIFT 音频** 和 **同步时钟** 两个职能，
同步模式下 GP2 被改为时钟 I/O，且 `if (!syncMode) bmStartAudio();` 跳过鼓音频，
于是**任何同步模式下鼓/DRIFT 全哑**。

| 项目 | 旧 | 新 |
|---|---|---|
| SYNC_IN | GP2 | **GP8** |
| SYNC_OUT | GP2 | **GP8**（同一插座，方向开机菜单选）|
| 鼓音频启动 | `if (!syncMode) bmStartAudio();` | **无条件 `bmStartAudio();`** |
| 同步模式下鼓/DRIFT | 全哑 | **保持有声** |

- 物理改动：LCD RST 原占 GP8，已跳线到 **3V3** 腾出 GP8。
- 开机按住 **pad 14（SW14 = 物理 pad 14 = GP9）** 出同步菜单：**pad 1 = Sync In**、**pad 2 = Sync Out**。
- 开机确认文案已改为 "external clock on GP8" / "clock output on GP8"。
- 该模式不存 EEPROM，每次开机手势决定；鼓机屏幕会显示 "SYNC"。

## 2. 声音方案（二选一，各烧各的 UF2）

### A. 原版声音 — `Firmware/Acid_Drip_V5_Original_Sound/`
- = git HEAD 主 `.ino` + MADCTL `0x40` 显示修复（显示修复是早前改动、从未提交，HEAD 里是 `0x80`）+ 其余 11 个文件照抄。
- 已编译：`Firmware/build_out_orig/Acid_Drip_V5_Original_Sound.ino.uf2`
  （Flash 151916B / 7%，RAM 56000B / 21%）

### B. AcidBox 303 移植版 — `Firmware/Acid_Drip_V5_Drum_Acid_Drift/`
- 主 `.ino` 内嵌 AcidBox 303 引擎移植（synthVoice / rosic TeeBee 滤波器 / overdrive / 波形表）。
- 已编译：`Firmware/build_out/Acid_Drip_V5_Drum_Acid_Drift.ino.uf2`
  （Flash 153004B，RAM 70048B）

**编码注意**：原版声音 `.ino` 注释里的破折号是字面乱码 `鈥?`（U+9225 + `?`），
303 版是标准 `—`（U+2014）。编辑时须按各文件实际字节匹配，不可互相套用。

## 3. 音频引脚分配（最终状态）

| 信号 | 引脚 | 说明 |
|---|---|---|
| ACID 音频 | **GP15** | Mozzi `MOZZI_AUDIO_PIN_1 15`，100R + 10nF RC 后到 3.5mm |
| 鼓 + DRIFT 音频 | **GP2** | `PWMAudio bmPWM(2)`；DRIFT 加性混入鼓缓冲，470R 后到 3.5mm |
| 同步 IN/OUT | **GP8** | 同一插座，方向由开机菜单选择 |
| TFT RST | 3V3 | 从 GP8 改跳 3V3 |
| 混合 | 物理电位器 | PCB 上合成 GP15 vs GP2；电位器拧到 ACID 端会**短路 GP2** |

## 4. 排查结论：MIX EDIT 里"只剩 ACID 有声"

1. **鼓机没启动** → 需按 **9+10** 切到鼓模式，再 **1+2** 播放。
2. **DRIFT 默认关** → 需按 **9+10+11**（0.5s）切到 DRIFT。
3. **物理混合电位器**拧在 ACID 端 → 短路 GP2，鼓声直接消失。
4. **MIX EDIT 防误触** → 旋钮必须"动一下"才接管（阈值 24 计数），退出存 EEPROM。

30 秒验证清单：9+10 → 1+2 ［PLAY］；物理电位器转离 ACID 端；MIX EDIT 里把 RES/DECAY 往右拧。

## 5. 文档更新

- **README.md**：硬件表同步行、接线块、同步章节整段重写为 GP8 + "同步不再静音鼓声"。
- **Acid_Drip_Quick_Reference_Guide_GP8.pdf**：重新生成的修正版速查卡（原 PDF 同步章节为旧版 GP2 描述）。
  - 生成脚本：`C:\Users\Rose\AppData\Local\Temp\opencode\make_qr_pdf.py`（fpdf2 生成）。

## 6. 文件清单（本会话产出）

```
Firmware/Acid_Drip_V5_Original_Sound/          # 原版声音工程（12 个文件，同步=GP8）
Firmware/Acid_Drip_V5_Drum_Acid_Drift/         # 303 AcidBox 版工程（同步=GP8）
Firmware/build_out_orig/Acid_Drip_V5_Original_Sound.ino.uf2
Firmware/build_out/Acid_Drip_V5_Drum_Acid_Drift.ino.uf2
README.md                                       # 已更新
Acid_Drip_Quick_Reference_Guide_GP8.pdf         # 新增
LOCAL_MODIFICATIONS.md                          # 本文件
```

## 7. 提交状态

- HEAD `3738a4a` 是原始上传版（MADCTL `0x80`、同步=GP2）。
- 本清单涉及的所有改动均**未提交**（工作区含：V5 两个工程的 .ino 改动、README、
  V5_Original_Sound 新目录、build 产物、本 PDF）。注意 git status 里 V4 工程与
  若干 sample/kicad 文件的 "M" 多为**行尾（LF/CRLF）归一化**差异，非内容改动。