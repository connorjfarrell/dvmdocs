# DVM-V1 Duplex Modem

The **DVM-V1** is the official DVMProject duplex modem. It is a derivative of the
[Multi-Mode Digital Voice Modem (MMDVM)](https://github.com/g4klx/MMDVM) design by Jon Naylor,
G4KLX, updated with **100% software calibration** — no trimpots. Paired with an analog radio
(or a separate Rx and Tx radio) and [`dvmhost`](../software/dvmhost/dvmhost.md), it turns
almost any analog radio system into a full-featured conventional or trunked digital repeater
for P25, DMR, and NXDN.

The board is STM32F4-based and runs [`dvmfirmware`](../software/dvmfirmware/dvmfirmware.md)
built with `Makefile.STM32F4_DVMV1`.

## Where to buy

Boards are available from the
[W3AXL Online Store](https://store.w3axl.com/products/dvm-v1-duplex-modem). Purchases support
DVMProject development.

## Hardware revisions

| Rev | Changes | Docs |
|-----|---------|------|
| **C.1** | First release. | [Schematics](https://github.com/DVMProject/dvmv1/blob/main/dvm-v1-c.1.pdf) · [Interactive BOM](https://htmlpreview.github.io/?https://github.com/DVMProject/dvmv1/blob/main/ibom-c.1.html) |
| **C.2** | Added M2.5 mounting holes; replaced the I2C header with an LED header. | [Schematics](https://github.com/DVMProject/dvmv1/blob/main/dvm-v1-c.2.pdf) · [Interactive BOM](https://htmlpreview.github.io/?https://github.com/DVMProject/dvmv1/blob/main/ibom-c.2.html) |

## Radio interfacing

The DVM-V1 connects to the radio through an **RJ45** jack carrying discriminator/flat audio in,
modulator audio out, PTT, and COS/ground. The exact pinout is in the
[interfacing diagram](https://github.com/DVMProject/dvmv1/blob/main/pics/interfacing.png) in
the repository.

For good digital performance the radio must pass a wider audio path than a typical 12.5 kHz
analog alignment allows — aim for an overall analog FM deviation of **2.75–2.83 kHz**. See the
[`dvmhost` calibration notes](../software/dvmhost/dvmhost.md#calibration).

## Using it with `dvmhost`

In `config.yml`, `system.modem.protocol.mode` should be `air`, with `protocol.type: uart` and
the correct `uart.port` (e.g. `/dev/ttyUSB0` or `/dev/ttyACM0`). Then run the
[`--setup` / `--cal` calibration procedure](../software/dvmhost/dvmhost.md#calibration). The
DVM-V1's software potentiometers are exposed under `system.modem.softpot`.

## Source / support

- Repository: <https://github.com/DVMProject/dvmv1>
- Discord: <https://discord.gg/3pBe8xgrEz>
