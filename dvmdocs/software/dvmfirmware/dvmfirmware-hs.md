# `dvmfirmware-hs` Hotspot Firmware

`dvmfirmware-hs` is the embedded firmware for low-power hotspot boards that use an **ADF7021**
RF transceiver IC driven by an STM32F103. It provides the raw RF interface for a dedicated-mode
DMR, P25, or NXDN hotspot. It is a direct fork of
[MMDVM_HS](https://github.com/juribeparada/MMDVM_HS).

```{note}
This firmware is **required** to interface a DVM hotspot with [`dvmhost`](../dvmhost/dvmhost.md).
NXDN support is experimental. A separate `usb-support` branch adds USB connectivity.
```

## Supported boards

A single Makefile, `Makefile.STM32FX`, targets generic STM32F103 + ADF7021 designs. The board
type is passed as an argument and selects the correct pin mapping — the hotspot boards share a
common design but pinouts differ, so choose the one matching your hardware (e.g.
`mmdvm-hs-hat-dual` for a duplex GPIO hat).

## Building

1. Install the ARM embedded toolchain (`arm-none-eabi-gcc` / `gcc-arm-none-eabi`).
2. Clone **with submodules**:
   ```bash
   git clone --recurse-submodules https://github.com/DVMProject/dvmfirmware-hs.git
   ```
3. Build, passing your board type:
   ```bash
   cd dvmfirmware-hs
   make -f Makefile.STM32FX mmdvm-hs-hat-dual
   ```

## Flashing via GPIO on a Raspberry Pi

```{warning}
Results vary between boards. These steps assume an MMDVM_HS-style board on the Pi GPIO header.
```

1. Free the GPIO serial port: remove `console=serial0,115200` from `/boot/cmdline.txt`, add
   `dtoverlay=disable-bt` to `/boot/config.txt`, and disable the serial getty. See
   [Raspberry Pi preparation](../dvmhost/installation.md#raspberry-pi-preparation). Reboot.
2. Install `stm32flash` (the distro package works fine): `sudo apt-get install stm32flash`.
3. `cd` to the build folder. Put a jumper across the **J1** points on the board — the red
   heartbeat LED should stop flashing (bootloader mode).
4. Flash (sudo needed for GPIO):
   ```bash
   # Raspberry Pi OS Bullseye and earlier:
   sudo stm32flash -v -w dvm-firmware-hs_f1.bin -i 20,-21,21,-20 -R /dev/ttyAMA0

   # Raspberry Pi OS Bookworm (Debian 12) and later — GPIO chip numbering changed:
   sudo stm32flash -v -w dvm-firmware-hs_f1.bin -i 532,-533,533,-520 -R /dev/ttyAMA0
   ```
   Success ends with `Wrote and verified address 0x0800…` and `Reset done.`
5. Remove the J1 jumper.

USB-connected boards use the `usb-support` branch and are flashed with the vendor's DFU tool.

## Tuning

Bandwidth, AFC, gain mode, and Rx/Tx frequency offsets for ADF7021 hotspots are set on the
`dvmhost` side under `system.modem.hotspot` in `config.yml` (`dmrDiscBWAdj`, `p25PostBWAdj`,
`adfGainMode`, `afcEnable`, `txTuning`, `rxTuning`, …). See
[`dvmhost` calibration](../dvmhost/dvmhost.md#hotspot-calibration-with-a-service-monitor).

## Source

<https://github.com/DVMProject/dvmfirmware-hs>
