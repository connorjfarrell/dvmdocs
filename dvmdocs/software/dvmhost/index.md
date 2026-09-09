# DVMHost Application Suite

[DVMHost](https://github.com/DVMProject/dvmhost) is the core software repository. A single
CMake build produces the primary applications used to create and connect digital radio
networks. Most of them share the [`fnecore`](https://github.com/DVMProject/fnecore) networking
library.

## Core applications

| Binary | Purpose | Page |
|--------|---------|------|
| `dvmhost` | Primary host — connects to a DVM modem (air interface or P25 V.24/DFSI) and processes DMR / P25 / NXDN traffic for a repeater, hotspot, or trunking channel. | [dvmhost](dvmhost.md) |
| `dvmfne` | Fixed Network Equipment — a routing core that interconnects `dvmhost` instances, consoles, bridges, and other FNEs. | [dvmfne](dvmfne.md) |
| `dvmbridge` | Analog/PCM audio bridge — connects analog or PCM audio (AllStar, USRP, line audio) to an FNE with realtime vocoding. | [dvmbridge](dvmbridge.md) |
| `dvmpatch` | Talkgroup patching utility — manually patches talkgroups of the same digital mode together. | [Supplementary applications](supplementary.md) |
| `dvmcmd` | Command-line client for sending remote-control commands to a `dvmhost` or `dvmfne` REST API. | [dvmcmd](dvmcmd.md) |

## Supplementary / monitoring applications

`dvmmon`, `sysview`, `tged`, and `peered` are TUI utilities for monitoring and editing
configuration. They are only built when project-wide TUI support is enabled. See
[Supplementary `dvmhost` Applications](supplementary.md).

## Next steps

- [Installation](installation.md) — dependencies, building, cross-compiling, Raspberry Pi prep.
- [`dvmhost`](dvmhost.md) — configuration and calibration.
- [`dvmfne`](dvmfne.md) — network core configuration.

## Source

- Repository & README: <https://github.com/DVMProject/dvmhost>
- Example configs: <https://github.com/DVMProject/dvmhost/tree/master/configs>
- Technical notes: <https://github.com/DVMProject/dvmhost/tree/master/docs>
