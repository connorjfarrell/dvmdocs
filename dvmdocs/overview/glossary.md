# Glossary

Terms and abbreviations used throughout DVMProject documentation and configuration files.

```{glossary}
Air interface
  The over-the-air RF protocol between a radio and a site. DVM implements the P25, DMR, and
  NXDN air interfaces. Contrast with the {term}`DFSI` wireline interface.

BER
  Bit Error Rate. The primary measurement used when
  [calibrating a modem](../software/dvmhost/dvmhost.md#calibration). Lower is better.

CC
  Control Channel. In a trunked system, the channel that carries signalling and directs
  radios to {term}`VC`s. May be *dedicated* (carries only signalling) or *non-dedicated*
  (interleaves signalling with voice/idle).

Color Code
  A 4-bit value (0–15) that identifies a DMR system, analogous to a CTCSS tone. Radios ignore
  traffic with a non-matching color code. Set with `system.config.colorCode`.

DFSI
  Digital Fixed Station Interface (TIA-102.BAHA). A wireline (non-RF) protocol for carrying
  P25 between a console/network and a base station, typically over V.24 synchronous serial or
  UDP. Used by the [DVM-V24](../hardware/dvm-v24.md) in `dfsi` modem mode.

DVRS
  Digital Vehicular Repeater System — a low-power mobile repeater. In DVM terms, a simplex or
  low-power hotspot rather than a full duplex site.

FNE
  Fixed Network Equipment. The [`dvmfne`](../software/dvmhost/dvmfne.md) routing core that
  interconnects hosts, consoles, and bridges, and links to other FNEs.

Hotspot
  A low-power personal access point. DVM hotspots use an {term}`ADF7021` transceiver and run
  [`dvmfirmware-hs`](../software/dvmfirmware/dvmfirmware-hs.md).

ADF7021
  The Analog Devices narrow-band ISM transceiver IC used by MMDVM-style hotspot boards.

iden table
  The channel identity / bandplan table (`iden_table.dat`) that maps a channel identity +
  channel number to an actual RF frequency. See [Channel identity tables](../guides/iden-tables.md).

LC
  Link Control. Signalling embedded in a DMR voice superframe (source/destination, flags).

LLA
  Link Layer Authentication. A P25 challenge-response scheme authenticating a radio to a
  site before registration. Uses the AES key in `system.config.secure.key`.

MMDVM
  Multi-Mode Digital Voice Modem — the original open-source modem project by Jon Naylor
  (G4KLX) that DVMProject firmware and the DVM-V1 are derived from.

NAC
  Network Access Code. A 12-bit value (hex, e.g. `293`) identifying a P25 system on a
  channel, analogous to a DMR color code. Set with `system.config.nac`.

Parrot
  A test talkgroup that records a transmission and immediately plays it back, letting a user
  check their own audio and signal. TGID 9990 by convention.

Peer ID
  A unique numeric identifier for anything that connects to an {term}`FNE` — a host, bridge,
  console, or another FNE. Set with `network.id` (host) or `master.peerId` / `peers[].peerId`
  (FNE).

RAN
  Radio Access Number. The NXDN equivalent of a {term}`NAC` / color code. Set with
  `system.config.ran`.

REST API
  The HTTP control/telemetry interface exposed by `dvmhost` and `dvmfne`, used by
  [`dvmcmd`](../software/dvmhost/dvmcmd.md), [`dvmprov`](../software/dvmprov.md), `pydvm`, and
  monitoring tools.

RFSS
  RF Sub-System. A P25 grouping of one or more sites under an {term}`FNE` / zone controller.
  Set with `system.config.rfssId`.

RID
  Radio ID (also *SU ID*, source ID). The unique numeric identity of a subscriber unit.
  Access is controlled by `rid_acl.dat`.

RPC
  Remote Procedure Call. The internal control channel used between a trunking {term}`CC` host
  and its {term}`VC` hosts (`system.config.controlCh` / `voiceChNo`). Distinct from the REST API.

SNDCP
  Sub-Network Dependent Convergence Protocol. P25 packet-data transport; used by the FNE
  `vtun` virtual tunnel feature.

TG / TGID
  Talkgroup / Talkgroup ID. A numeric group address that radios affiliate to and select.
  Routing and access are defined in `talkgroup_rules.yml`.

TSBK
  Trunking Single Block. A P25 control-channel signalling message (grants, affiliations,
  announcements).

TSCC
  Timeslot Control Channel. The DMR (Tier III–style) trunking control channel, which occupies
  one timeslot. Enabled under `protocols.dmr.control`.

VC
  Voice Channel. In a trunked system, a channel assigned on demand by the {term}`CC` to carry
  a call.

WACN
  Wide Area Communications Network identifier. A 20-bit value (hex, e.g. `BB800`) that, with
  the {term}`System ID`, uniquely identifies a P25 network. Set with `system.config.netId`.

System ID
  A 12-bit P25 value identifying a system within a {term}`WACN`. Set with `system.config.sysId`.
```
