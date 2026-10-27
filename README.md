# Personal fork – 200 MS/s + start trigger

This branch (`200msps`) is a personal modification of
[pico-coder/sigrok-pico](https://github.com/pico-coder/sigrok-pico).
The original README follows below.

## Changes and why

| Change | File | Why |
|---|---|---|
| System clock 200 MHz (`SYS_CLK_MHZ=200`, plus PLL settings for RP2350) | `CMakeLists.txt` | Stock firmware runs at 125 MHz. At 200 MHz: true, evenly spaced 200 MS/s (5 ns). |
| Sample rate limit 120 → 200 MS/s | `sr_device.c` | Firmware rejected anything above 120 MS/s. |
| Clamp oversized captures above 20 MS/s | `pico_sdk_sigrok.c` | Requests larger than RAM switched to streaming, which overflows at high rates and aborted after half a buffer. Now the request is cut to what fits, giving the full buffer in one shot. |
| Pull-downs off on digital inputs | `pico_sdk_sigrok.c` | Inputs become high impedance (only pin capacitance + leakage). |
| Start trigger on GP22 | `pico_sdk_sigrok.c` | Capture waits until GP22 is high, then starts. GP22 has a pull-up, so unconnected = starts immediately (original behavior). |
| `PICO_MODE 0` (baseline) | `sr_device.h` | D2–D22 + analog A0–A2 + debug UART. |

### Clock details
- **RP2040:** 200 MHz is officially supported since Pico SDK 2.1.1; the SDK raises the core voltage to 1.15 V automatically.
- **RP2350:** rated 150 MHz, so 200 MHz is a mild overclock at default voltage.

### Start trigger details
- The CPU waits right before the ADC and PIO are started together, so analog/digital alignment is unchanged in every mode.
- **Level-sensitive:** hold GP22 low before pressing Run; the capture starts when it goes high.
- Start jitter ~0.1–0.2 µs (occasionally a few µs due to interrupts).
- Stop in PulseView ends the wait cleanly.
- D22 is reserved for the trigger; don't use it as a normal input.
- To disable it (D22 back to a normal input), set `#define TRIG_EN 0` in `pico_sdk_sigrok.c`.

## Build

Requires Pico SDK **≥ 2.1.1**, with submodules (TinyUSB):

```bash
git -C $PICO_SDK_PATH submodule update --init

cd pico_sdk_sigrok
# RP2040 (Pico)
mkdir -p build && cd build
cmake .. -DPICO_BOARD=pico -DPICO_SDK_PATH=$PICO_SDK_PATH && make -j$(nproc)
cd ..
# RP2350 (Pico 2), from pico_sdk_sigrok/
mkdir -p build_pico2 && cd build_pico2
cmake .. -DPICO_BOARD=pico2 -DPICO_SDK_PATH=$PICO_SDK_PATH && make -j$(nproc)
```

Flash: hold BOOTSEL, plug in, copy `pico_sdk_sigrok.uf2`.

## Verify 200 MHz

1. Debug UART (GP0, 115200 baud) prints `clk_sys = 200000kHz` at startup.
2. Capture a clock of known frequency and measure its average period: it must match (e.g. 10 MHz → 100 ns). If the clock change didn’t apply, periods read 1.6× too long.

## Usage notes

- **Pins (mode 0):** D2 = GP2 … D21 = GP21, A0–A2 = GP26–28, GP22 = trigger, GP0/GP1 = debug UART.
- **Fastest mode:** D2–D5 only, no analog; select 200 or 100 MHz.
- Digital channels must be **contiguous from D2** (driver rule); analog channels can be any combination.
- Select a large sample count (e.g. 1M); the firmware clamps it to the buffer:

| Board | Buffer | 200 MS/s, D2–D5 |
|---|---|---|
| Pico (RP2040) | 220 KB | ~440K samples ≈ 2.2 ms |
| Pico 2 (RP2350) | 476 KB | ~951K samples ≈ 4.8 ms |

- With any analog channel on, everything runs at ADC speed (max 500 kS/s shared across analog channels).
- PulseView warnings `Sample rate above 120Msps` / `Non integer PIO divisor of 120Mhz` come from the PC driver, which assumes 120 MHz. Harmless at 200 and 100 MS/s.
- Inputs have no pull-down: enabled channels with nothing connected float and read random values. Disable unused channels or tie them to GND.

## Known limits

- **Streaming:** limited by USB full-speed (~0.5–1 MS/s for 4 busy channels); high rates only work as in-RAM captures.
- **Inputs:** 3.3 V maximum, no protection beyond the chip's own.

---
---
---
# sigrok-pico

Use a Raspberry Pi PICO (RP2040) as a logic analyzer and oscilloscope with sigrok.

## Status
=======
Please start with the Getting Started page : https://github.com/pico-coder/sigrok-pico/blob/main/GettingStarted.md

**Merged to mainline sigrok** (September 2023)

Install from [sigrok.org/downloads](https://sigrok.org/wiki/Downloads). PulseView 0.4.2 and sigrok-cli 0.7.2 do not support sigrok-pico.

## Quick Links

| Document | Description |
|----------|-------------|
| [USER_GUIDE.md](USER_GUIDE.md) | Getting started and analyzer operations |
| [TECHNICAL.md](TECHNICAL.md) | Serial protocol and build instructions |
| [pulseview/Readme.md](pulseview/Readme.md) | Windows installer |

## Directory Structure

```
sigrok-pico/
├── pico_pgen/              # Digital function generator for testing
├── pico_sdk_sigrok/        # RP2040 firmware (see release/ for UF2 files)
└── pulseview/              # Windows installer (unofficial)
```

## Overview

This project implements a sigrok driver for the Raspberry Pi PICO RP2040 using the PICO SDK CDC serial library. It works with both **PulseView** (GUI) and **sigrok-cli** (command-line):

- **21 digital channels** (D2-D22)
- **3 analog channels** (A0-A2)
- **Mixed-mode capture** (combined digital + analog)

### Using with PulseView

PulseView is the recommended graphical interface for sigrok-pico:

1. Install PulseView from [sigrok.org/downloads](https://sigrok.org/wiki/Downloads)
2. Flash the PICO with the appropriate UF2 firmware
3. In PulseView, select "raspberrypi_pico" driver and configure the serial port

> **Note**: PulseView 0.4.2 does not support sigrok-pico. Use a newer version or the [unofficial Windows installer](pulseview/Readme.md).

## Firmware

### Precompiled

Pre-compiled UF2 files are available in [`pico_sdk_sigrok/release/`](pico_sdk_sigrok/release/):

| File | Description |
|------|-------------|
| pico_baseline.uf2 | Standard firmware |
| pico_dig26.uf2 | 26-channel digital |
| pico_dig32.uf2 | 32-channel digital |
| pico2_*.uf2 | PICO 2 variants |

### Build Your Own

Building is straightforward using the VSCode extension:

1. Install the "Raspberry Pi Pico Project" extension in VSCode
2. Import the project using the extension
3. Select the latest SDK version when prompted
4. Select the board type (e.g., `pico` or `pico2`)
5. Click "Run Project (USB)" to build and flash

For command-line builds and more details, see [TECHNICAL.md](TECHNICAL.md).

## License

See [LICENSE](LICENSE) for details.
