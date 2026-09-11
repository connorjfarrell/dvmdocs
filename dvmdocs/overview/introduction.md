# Introduction

The **Digital Voice Modem (DVM) Project** is an open-source effort to build and connect
digital land-mobile-radio (LMR) networks using commodity hardware and amateur-grade equipment.
It provides everything needed to stand up conventional or trunked digital repeater sites and to
link them together into wide-area networks.

## Supported digital modes

DVMProject software implements three common digital LMR air interfaces:

| Mode | Origin | Notes |
|------|--------|-------|
| **P25** (Project 25) | APCO / TIA-102 | Phase 1 FDMA (C4FM). Conventional and trunked. Also supports a TIA-102 V.24 / DFSI interface to commercial P25 equipment such as the Motorola Quantar. |
| **DMR** (Digital Mobile Radio) | ETSI TS 102 361 | 2-slot TDMA. Conventional and Tier III–style trunking via a Timeslot Control Channel (TSCC). |
| **NXDN** | Icom / Kenwood | 4-FSK FDMA. Conventional and trunked. Support is considered **experimental**. |

A single DVM host can run one mode at a time (fixed mode) or switch between modes automatically
(mixed / multi-mode operation).

## The two halves of the project

- **Hardware** — in-house modem and adapter designs: the [DVM-V1](../hardware/dvm-v1.md) duplex
  modem for connecting analog repeaters/base stations, the [DVM-V24](../hardware/dvm-v24.md)
  adapter for Motorola V.24 synchronous serial equipment, and support for MMDVM-style ADF7021
  hotspot boards. See [Supported Hardware](supported-hardware.md).
- **Software** — the [`dvmhost` application suite](../software/dvmhost/index.md) (host, FNE,
  bridge, patch, and command-line tools), the [modem](../software/dvmfirmware/dvmfirmware.md)
  and [hotspot](../software/dvmfirmware/dvmfirmware-hs.md) firmware forks, the
  [Desktop Dispatch Console](../software/dvmconsole.md), and provisioning/monitoring utilities.

## Conventional vs. trunked

- **Conventional** — each channel is used directly; users select a channel/talkgroup manually.
  DVM can optionally send P25 control-channel messaging on a conventional channel for features
  like affiliation and channel grants.
- **Trunked** — a control channel (CC) directs radios to voice channels (VCs) on demand. DVM
  supports dedicated-CC trunking sites with one or more voice channels, and multi-site trunking
  when sites are linked through an [FNE](../software/dvmhost/dvmfne.md).

## Usage policy and restrictions

```{warning}
**DVMProject components must never be used for public safety communications or life safety
systems**, where a failure could contribute to injury or loss of life — not even as a backup
or auxiliary path.
```

- The software is licensed under the **GPL-2.0** and is intended for **amateur radio and
  educational use only**. Commercial, professional, and governmental use is strongly
  discouraged, unsupported, and disclaimed by the authors.
- The software has **not** undergone mission-critical validation and carries no warranty as to
  reliability, uptime, or fault tolerance.
- **Passive, receive-only hobbyist monitoring** of publicly available radio traffic is outside
  the scope of the restriction.
- Requesting help with prohibited use cases in official DVMProject channels (Discord, issue
  trackers) may result in removal and refusal of support.

By using this software or participating in the DVMProject community you agree to these terms.
The full policy is published as
[`usage_guidelines.md`](https://github.com/DVMProject/dvmhost/blob/master/usage_guidelines.md)
in the `dvmhost` repository.

## Getting help

- **Discord:** <https://discord.gg/3pBe8xgrEz>
- **Source & issues:** <https://github.com/DVMProject>

## Project repositories

| Repository | Purpose |
|------------|---------|
| [`dvmhost`](https://github.com/DVMProject/dvmhost) | Core application suite (`dvmhost`, `dvmfne`, `dvmbridge`, `dvmpatch`, `dvmcmd`) and technical documentation. |
| [`dvmfirmware`](https://github.com/DVMProject/dvmfirmware) | Modem firmware (STM32F4 / Arduino Due). |
| [`dvmfirmware-hs`](https://github.com/DVMProject/dvmfirmware-hs) | Hotspot firmware (STM32F1 + ADF7021). |
| [`dvmv1`](https://github.com/DVMProject/dvmv1) | DVM-V1 duplex modem hardware. |
| [`dvmv24`](https://github.com/DVMProject/dvmv24) | DVM-V24 Motorola V.24 USB adapter hardware/firmware. |
| [`dvmconsole`](https://github.com/DVMProject/dvmconsole) | Desktop Dispatch Console. |
| [`dvmprov`](https://github.com/DVMProject/dvmprov) | FNE provisioning web tool (archived). |
| [`dvmbridge` helpers](https://github.com/DVMProject/dvmusrp) · [`pydvm`](https://github.com/DVMProject/pydvm) · [`dvmvocoder`](https://github.com/DVMProject/dvmvocoder) · [`fnecore`](https://github.com/DVMProject/fnecore) · [`iden-channel-calculator`](https://github.com/DVMProject/iden-channel-calculator) | Supporting libraries and utilities. |

The full organization is at <https://github.com/DVMProject>.
