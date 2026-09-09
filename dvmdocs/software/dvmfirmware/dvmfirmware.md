# `dvmfirmware` Duplex Modem Firmware

`dvmfirmware` is the embedded firmware for full-power modem boards. It connects directly to the
air interface (usually an analog radio) and performs the actual reception and generation of the
digital waveforms for DMR, P25, and NXDN. It is a direct fork of
[MMDVM](https://github.com/g4klx/MMDVM) by Jon Naylor, G4KLX.

```{note}
This firmware is **required** to interface a DVM modem with [`dvmhost`](../dvmhost/dvmhost.md).
NXDN support is experimental.
```

## Supported boards

Build with `make -f <Makefile>` for your board:

| Board | Makefile |
|-------|----------|
| Arduino Due | `Makefile.SAM3X8_DUE` |
| Generic STM32F4 | `Makefile.STM32F4` |
| RepeaterBuilder STM32-DVM (POG) | `Makefile.STM32F4_POG` |
| WA0EDA "v3" STM32F4 405/446 (MTR2000, MASTR III) | `Makefile.STM32F4_EDA` |
| DVMProject [DVM-V1](../../hardware/dvm-v1.md) | `Makefile.STM32F4_DVMV1` |

## Building

Compile on Linux (other platforms: YMMV).

### STM32F4 boards

1. Install the ARM embedded toolchain (`arm-none-eabi-gcc` / `gcc-arm-none-eabi`).
2. Clone **with submodules** — the STM32 platform files are submodules:
   ```bash
   git clone --recurse-submodules https://github.com/DVMProject/dvmfirmware.git
   ```
3. Build:
   ```bash
   cd dvmfirmware
   make -f Makefile.STM32F4_DVMV1        # or the Makefile for your board
   ```

### Arduino Due

1. Install the Arduino SDK under `~/.arduino15`.
2. Edit `~/.arduino15/packages/arduino/hardware/sam/1.6.8/platform.txt` and append the
   Cortex-M3 math library to the `recipe.c.combine.pattern` linker line
   (`.../CMSIS/CMSIS/Lib/GCC/libarm_cortexM3l_math.a`) — see the upstream README for the exact
   before/after.
3. Build with `make -f Makefile.SAM3X8_DUE`.

## Flashing

Flashing methods vary by board and are mostly not documented upstream. In general:

- **USB / DFU boards** — use the vendor's DFU tool, or `dvmhost --boot` to reboot the modem
  into its bootloader first.
- **GPIO-attached STM32 on a Raspberry Pi** — free the serial port first (see
  [Raspberry Pi preparation](../dvmhost/installation.md#raspberry-pi-preparation)), then flash
  over `/dev/ttyAMA0` with `stm32flash` (see the
  [hotspot firmware page](dvmfirmware-hs.md#flashing-via-gpio-on-a-raspberry-pi) for the
  `stm32flash` invocation pattern).
- **ST-Link / SWD** — `st-flash write dvm-firmware_f4.bin 0x8000000`.

## Modem configuration area

The modem stores some settings in a configuration area on the board. `dvmhost` normally reads
these; set `system.modem.ignoreModemConfigArea: true` in `config.yml` to have `dvmhost` ignore
them and use the values from `config.yml` exclusively.

## Source

<https://github.com/DVMProject/dvmfirmware>
