# OpenGUTv1 Firmware

Firmware for OpenGUTv1: nRF52840-based platform built on Zephyr (NCS).

This project implements:
- Stereo audio capture from a PDM microphone.
- WAV playback from microSD through I2S.
- Live microphone monitoring (PDM input to I2S output).
- Button-driven mode selection and execution.
- 4-LED status UI for mode indication, progress, and error states.

## Features

### Audio pipeline
- Records **16-bit stereo WAV** at **16 kHz** to SD card.
- Plays WAV files from SD card over I2S.
- Supports mono or stereo WAV input for playback (mono is duplicated to both channels).
- Live monitor mode streams microphone input directly to I2S output in real time.

### Storage
- Uses SPI-connected microSD card (`/SD:` mount point, FAT filesystem).
- Automatically writes `hello.txt` to verify SD write path.
- Rotating record filenames: `rec0.wav` ... `rec9.wav` (first missing index is used; wraps to `rec0.wav`).

### UI behavior
- **SW1**: mode select
  - Short press: next mode
  - Long press (>= 700 ms): previous mode
- **SW2**: execute selected mode (and stop recording/live monitor)
- **LED1..LED4** indicate current mode; active mode blinks while running.
- Error condition: selected mode LED blinks continuously.

### Modes
- **Mode 1 (LED1): Record Stereo**
  - Starts recording to `/SD:/recX.wav`
  - Press SW2 again to stop
- **Mode 2 (LED2): Live Monitor**
  - Streams PDM mic to I2S output
  - Press SW2 again to stop
- **Mode 3 (LED3): Play Loud**
  - Plays `/SD:/loud.wav`
- **Mode 4 (LED4): Play Medium**
  - Plays `/SD:/medium.wav`

## Hardware Mapping (from DeviceTree overlay)

### GPIO and controls
- LED1: `P1.14` (active low)
- LED2: `P1.13` (active low)
- LED3: `P1.12` (active low)
- LED4: `P1.11` (active low)
- SW1: `P1.06` (pull-up, active low)
- SW2: `P0.28` (pull-up, active low)

### Audio + SD interfaces
- PDM CLK: `P0.21`
- PDM DIN: `P1.08`
- I2S SCK: `P0.12`
- I2S LRCK: `P0.20`
- I2S SDOUT: `P0.15`
- SPI1 SCK: `P0.14`
- SPI1 MOSI: `P0.27`
- SPI1 MISO: `P0.26`
- SD CS: `P0.16`

## Prerequisites

Install one of these workflows:
- **nRF Connect for Desktop + Toolchain Manager + VS Code extension**, or
- **West CLI + Zephyr/NCS environment**.

Typical required tools:
- `west`
- `cmake`
- `ninja`
- ARM GCC toolchain
- One flash/debug tool: `nrfjprog`, `JLink`, or `pyocd`

## Build

From repository root:

```bash
west build -b nrf52840dk_nrf52840 -p always
```

Firmware artifact is generated at:
- `build/zephyr/zephyr.hex`

## Upload Firmware to PCB

You can flash either:
1. An **nRF52840 DK** directly, or
2. A **custom PCB** over SWD using an external probe.

### Option A: Flash nRF52840 DK directly

Connect the board over USB, then run:

```bash
west flash
```

If multiple probes are connected, specify one (example):

```bash
west flash --runner nrfjprog --dev-id <probe_serial>
```

### Option B: Flash custom PCB via SWD (recommended for production PCB)

1. Connect a debug probe (J-Link, nRF52840 DK used as debugger, or CMSIS-DAP) to your PCB:
   - `SWDIO` -> MCU SWDIO
   - `SWCLK` -> MCU SWDCLK
   - `GND` -> GND
   - `VTref`/`VCC` -> target voltage reference
   - Optional: `RESET` -> nRESET
2. Power your PCB correctly (battery/regulator/external supply as designed).
3. Build firmware if not already built.
4. Program the HEX file.

Using `nrfjprog`:

```bash
nrfjprog --eraseall
nrfjprog --program build/zephyr/zephyr.hex --verify
nrfjprog --reset
```

Using `west` with nrfjprog runner:

```bash
west flash --runner nrfjprog --hex-file build/zephyr/zephyr.hex
```

Using pyOCD (if your probe/target config supports it):

```bash
pyocd flash build/zephyr/zephyr.hex --target nrf52840
```

### Flashing checklist for custom PCB
- SWD wiring continuity is correct.
- Probe detects the target voltage.
- nRF52 is not held in reset by external circuitry.
- Boot/config pins and power rails are stable.
- SD card is FAT-formatted and inserted before running SD/audio modes.

## Runtime Notes

- On first boot, firmware may configure nRF52840 `UICR.REGOUT0` for 3.0 V output and reset.
- Logging is disabled in this configuration (`CONFIG_LOG=n`).
- Keep `loud.wav` and `medium.wav` in SD root for playback modes.

## Repository Structure

```text
src/main.c        Application state machine and mode logic
src/PDM.c         PDM capture and live monitor
src/Playback.c    WAV playback over I2S
src/SD.c          SD/FATFS initialization and file helpers
src/led.c         LED driver helpers
src/swicthes.c    Button handling (debounced interrupts)
prj.conf          Zephyr Kconfig
nrf52840dk_nrf52840.overlay  DeviceTree pin/peripheral mapping
```

## Troubleshooting

- `SD_Init` fails:
  - Check SPI wiring, CS pin, SD power, and FAT formatting.
- Playback mode exits quickly:
  - Confirm `/SD:/loud.wav` and `/SD:/medium.wav` exist and are valid 16-bit PCM WAV.
- No audio output in monitor/playback:
  - Verify I2S wiring (`SCK`, `LRCK`, `SDOUT`) and codec/amplifier clocks.
- Mode buttons not responding:
  - Confirm SW1/SW2 pin mapping and pull-up behavior.

## License

Add your project license here (for example MIT, Apache-2.0, proprietary).
