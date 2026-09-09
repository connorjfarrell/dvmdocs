# Supported Hardware

`dvmhost` talks to the air interface through a **modem**: either an in-house DVM board, an
MMDVM-derived board running the DVMProject firmware forks, or the DVM-V24 adapter for
commercial P25 equipment. This page summarizes what is known to work.

```{note}
It is **required** to run the DVMProject [firmware forks](../software/dvmfirmware/index.md)
(`dvmfirmware` / `dvmfirmware-hs`), not stock MMDVM firmware, to interface a modem or hotspot
with `dvmhost`.
```

## Hotspots

Low-power boards based on the MMDVM duplex/simplex hotspot design using the **ADF7021** RF
transceiver IC, driven by an STM32F1 microcontroller. These run
[`dvmfirmware-hs`](../software/dvmfirmware/dvmfirmware-hs.md) and support DMR, P25, and
(experimental) NXDN in simplex or duplex.

Boards that are generally compatible include generic STM32F103 + ADF7021 designs, the various
`MMDVM_HS` / `MMDVM_HS_Dual_Hat` style boards, and RepeaterBuilder hotspot boards. Because
"the hotspot boards are loosely designed around a common factor but not all are created
equally," pinouts and tuning vary between boards — pick the matching board type when building
the firmware.

## Modems

Full-power modems that provide analog (discriminator/line) interfaces to radio base-station
equipment. These run [`dvmfirmware`](../software/dvmfirmware/dvmfirmware.md).

### ✅ RECOMMENDED: DVM-V1 Duplex Modem

The official DVMProject duplex modem. STM32F4-based, 100% software-calibrated, connects to
radios over RJ45.

- Store: [W3AXL Online Store](https://store.w3axl.com/products/dvm-v1-duplex-modem)
- Hardware info: [DVM-V1 Duplex Modem](../hardware/dvm-v1.md)
- Repository: <https://github.com/DVMProject/dvmv1>

### Other MMDVM-style modem boards

Generic STM32F4 modem boards, RepeaterBuilder STM32 (POG) boards, WA0EDA STM32F4 (405/446)
boards, and Arduino Due–based boards are supported by `dvmfirmware` via dedicated build
targets. See the [firmware page](../software/dvmfirmware/dvmfirmware.md) for the board/Makefile
matrix.

## V.24 / DFSI equipment

### ✅ RECOMMENDED: DVM-V24 Synchronous V.24 to USB Adapter

An in-house design that adapts the legacy Motorola V.24 synchronous serial interface used by
base stations and comparators (Quantar, AstroTAC) to `dvmhost`. **P25 only.** Firmware
revision 2.0 or greater is required for use with `dvmhost`.

- Store: [W3AXL Online Store](https://store.w3axl.com/products/dvm-v24-v2-usb-converter-for-v-24-equipment)
- Hardware info: [DVM-V24 Motorola V.24 USB Adapter](../hardware/dvm-v24.md)
- Repository: <https://github.com/DVMProject/dvmv24>

## Host computer

| Application | Minimum (known working) | Recommended |
|-------------|------------------------|-------------|
| `dvmhost` | Raspberry Pi 1B, 512 MB RAM, single-core | Raspberry Pi 3+ / thin client / x86_64, 1 GB+ RAM, dual/quad-core |
| `dvmfne` | x86_64 server, 2 GB RAM, quad-core | x86_64 server, 4 GB+ RAM, quad-core+ (scales with network size) |

```{note}
Cross-compilation support for the Raspberry Pi 1/2/3 was removed in DVM R05A02. Use a
Raspberry Pi 4 or newer (or an x86_64 host) for new installs.
```

## Source / further reading

- DVM-V1 hardware: <https://github.com/DVMProject/dvmv1>
- DVM-V24 hardware: <https://github.com/DVMProject/dvmv24>
- Modem firmware (supported board list): <https://github.com/DVMProject/dvmfirmware>
- Hotspot firmware: <https://github.com/DVMProject/dvmfirmware-hs>
- `dvmhost` hardware requirements: <https://github.com/DVMProject/dvmhost/blob/master/README.md#hardware-requirements>
- Upstream MMDVM project: <https://github.com/g4klx/MMDVM>
