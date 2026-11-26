# Licensing

This repository is a fork of **Acid Drip** by Marcus (lonesoulsurfer):
https://github.com/lonesoulsurfer/Acid_Drip_Bassline_and_Drum_Synth

The upstream licences apply to everything that came from that project. This fork's
changes are released under the same terms. The licence files are kept verbatim:

- [LICENSE](LICENSE) — GNU General Public License v3.0 (firmware)
- [LICENSE-HARDWARE](LICENSE-HARDWARE) — CERN Open Hardware Licence v2, Strongly Reciprocal (upstream hardware files)

## Firmware: GPL-3.0

The firmware is the code in the `Firmware/` folder.

- Licence: GNU General Public License v3.0. See [LICENSE](LICENSE).
- Copyright (c) 2026 Marcus (lonesoulsurfer).
- Modifications in this fork: Copyright (c) 2026 cxandy. See [Changes in this fork](#changes-in-this-fork).
- The acid synth engine is adapted from [Badass Bass](https://github.com/Tschibo00/BadAssbasSSynth) by Tschibo00. Badass Bass uses GPL-3.0. Thus the Acid Drip firmware must also use GPL-3.0.

You can use, change and share the firmware. If you share a changed version, you must also share its source code under GPL-3.0.

### Changes in this fork

Fork source: https://github.com/cxandy/Acid_Drip

Changes relative to upstream `f88e64d`:

- Sync clock pin selected **at boot** (GPIO8 on the AZSMZ AD-5, GPIO2 on the upstream original PCB) instead of a compile-time choice.
- LCD reset tied to 3V3 on the AZSMZ AD-5, freeing GPIO8; drums and DRIFT stay audible in both sync directions.
- `MADCTL 0x40` display-orientation fix.
- Acid filter switched back to the older sound profile.
- Second sketch under `Firmware/Acid_Drip_V5_AcidBox/` with an AcidBox 303 engine port.

The full list, including the reasoning and the known issues, is in [README.md](README.md). Each modified source file carries a short modification notice at the top.

GPL-3.0 compliance for the assembled units:

- The complete corresponding source is this public repository.
- Build instructions are in [README.md](README.md); CI builds both sketches.
- The firmware is flashed by holding BOOTSEL and dragging a `.uf2` file. This is not locked or signed, so the owner can install a modified build (GPLv3 §6, Installation Information).
- The licence text is included with every unit, together with the source URL.

## Hardware: CERN-OHL-S-2.0

The hardware files that came from upstream — the KiCad project, front panel files, Gerber files and schematic in this repository — are licensed under the CERN Open Hardware Licence Version 2, Strongly Reciprocal.

- Licence: [LICENSE-HARDWARE](LICENSE-HARDWARE).
- Copyright (c) 2026 Marcus (lonesoulsurfer).

You can make, change and sell boards from these files. If you share or sell a changed design, you must also share the changed design files under the same licence.

**Not covered by CERN-OHL-S:** the assembled instrument sold by this fork's author uses a different board, front panel and enclosure, drawn from scratch in JLCEDA from an empty project. Those design files are not part of this repository and are not derived from the upstream KiCad or panel files. They remain the author's copyright. The upstream hardware files in this repository are kept for reference and stay under CERN-OHL-S-2.0.

## Name and trademark

Neither licence grants rights to the name **Acid Drip**. GPLv3 §7 does not grant trademark rights.

- Allowed: describing the firmware or product as *based on Acid Drip by Marcus (lonesoulsurfer)*, or as *Acid Drip–compatible*, with clear credit.
- Not allowed without written permission: presenting the product itself as *Acid Drip*, *Acid Drip V5*, or as an official product of the original author.

## Code from other projects

### DrumKid

The drum engine is adapted from [DrumKid](https://github.com/mattybrad/drumkid). DrumKid uses the MIT License. The MIT License lets other projects use DrumKid code, and it is compatible with GPL-3.0. The MIT License requires this notice:

```
MIT License

Copyright (c) 2019 Matt Bradshaw

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

### Libraries

This repository does not include a copy of these libraries. The Arduino Library Manager installs them.

| Library | Licence |
|---------|---------|
| [Mozzi](https://github.com/sensorium/Mozzi) (version 2) | LGPL-2.1 |
| [Adafruit GFX Library](https://github.com/adafruit/Adafruit-GFX-Library) | BSD |
| [Adafruit ILI9341](https://github.com/adafruit/Adafruit_ILI9341) | MIT |
| [Arduino-Pico core](https://github.com/earlephilhower/arduino-pico) | LGPL-2.1 |

The `.uf2` files in the releases include compiled code from these libraries.
