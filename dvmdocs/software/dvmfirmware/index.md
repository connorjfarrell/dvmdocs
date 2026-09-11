# DVMFirmware Hotspot & Modem Firmware

DVMProject maintains heavily rewritten forks of the original MMDVM hotspot and modem firmware.
**These forks are required** to interface DVM modems and hotspots with
[`dvmhost`](../dvmhost/dvmhost.md); stock MMDVM firmware is not compatible with the DVM modem
protocol.

Both forks support DMR, P25, and NXDN. NXDN support is considered **experimental**.

## `dvmfirmware` — modem firmware

For STM32F4- or Arduino Due–based full-power modem boards, including the
[DVM-V1](../../hardware/dvm-v1.md), RepeaterBuilder STM32 (POG), WA0EDA, and generic STM32F4
boards.

[Read more](dvmfirmware.md)

## `dvmfirmware-hs` — hotspot firmware

For STM32F1 hotspot boards using the ADF7021 RF transceiver IC (`MMDVM_HS` and derivatives).
Supports simplex and duplex hotspot boards.

[Read more](dvmfirmware-hs.md)

## Source

- Modem firmware: <https://github.com/DVMProject/dvmfirmware>
- Hotspot firmware: <https://github.com/DVMProject/dvmfirmware-hs>
