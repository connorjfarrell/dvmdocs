# DVMProject Hardware Overview

DVMProject maintains several in-house designs for interfacing with physical radio devices. See
[Supported Hardware](../overview/supported-hardware.md) for third-party boards that also work.

## DVM-V1

![DVM-V1 Photo](../media/hardware/dvm-v1.jpg)

The DVM-V1 is a derivative of the
[Multi-Mode Digital Voice Modem (MMDVM)](https://github.com/g4klx/MMDVM) design originally
created by Jon Naylor, G4KLX. The DVM-V1 modem allows conventional analog radio devices to
modulate and demodulate the C4FM / 4-FSK / TDMA digital signals used by P25, DMR, and NXDN. In
essence, any analog radio system can be converted to a full-featured conventional or trunked
digital repeater using nothing more than a single modem plus a repeater, duplex base station,
or any combination of receiver and transmitter. It is fully software-calibrated.

[Read more](dvm-v1.md)

## DVM-V24

![DVM-V24 Photo](../media/hardware/dvm-v24.jpg)

The DVM-V24 is a new in-house design that adapts the legacy Motorola V.24 synchronous serial
interface used by base stations and comparators like the Quantar and AstroTAC to the
[`dvmhost`](../software/dvmhost/index.md) application over USB. It is P25-only and requires
firmware revision 2.0 or newer. Two hardware revisions exist — the original V1 and the newer
V2, which moves USB↔serial onto a dedicated CP2102 to fix lockups.

[Read more](dvm-v24.md)
