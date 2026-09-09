# Example System Configurations

This page explains the basic types of digital radio systems that can be created with
DVMProject and links to the guides for installing and configuring each component.

The configurations begin with the simplest setup (a single conventional channel) and build
up. It's good practice to read through each section in order, since later multi-channel and
multi-site systems reuse the same building blocks.

All of these assume you have already
[built and installed `dvmhost`](../software/dvmhost/installation.md) and flashed the
appropriate [firmware](../software/dvmfirmware/index.md).

---

## Conventional mixed-mode single-channel system

A single channel that automatically switches between P25, DMR, and/or NXDN as traffic
arrives. The simplest possible DVM system and the best starting point.

### Requirements

- A [hotspot](supported-hardware.md#hotspots) or [modem](supported-hardware.md#modems)
- A Linux computer running [`dvmhost`](../software/dvmhost/dvmhost.md)
- A P25-, DMR-, and/or NXDN-capable radio

### Setup

1. Flash [`dvmfirmware`](../software/dvmfirmware/dvmfirmware.md) (modem) or
   [`dvmfirmware-hs`](../software/dvmfirmware/dvmfirmware-hs.md) (hotspot).
2. Connect the modem to the host (USB, or GPIO UART on a Raspberry Pi — see the
   [Pi preparation notes](../software/dvmhost/installation.md#raspberry-pi-preparation)).
3. For a modem, wire the discriminator/line audio and PTT to the repeater or Rx/Tx radios.

### Configuration

In `config.yml`:

- `protocols`: enable `dmr`, `p25`, and/or `nxdn`; leave all control-channel options disabled.
- `system.fixedMode: false` and `system.modeHang` to taste (mixed mode).
- `system.duplex`: `true` for a duplex modem/repeater, `false` for a simplex hotspot.
- `system.config`: set `channelId` / `channelNo` to an entry in `iden_table.dat` that yields
  your operating frequency, plus `colorCode` (DMR), `nac` (P25), `ran` (NXDN).
- `network.enable: false` for standalone, or point it at your [FNE](../software/dvmhost/dvmfne.md).

Then [calibrate the modem](../software/dvmhost/dvmhost.md#calibration).

---

## Conventional P25 single-channel system with control messaging

A single conventional P25 channel that also transmits TSBK control messaging, enabling
affiliation, channel-grant, and unit-registration behavior without a dedicated control
channel.

### Configuration

Starting from the conventional system above, in `config.yml` under `protocols.p25.control`:

- `enable: true`, `dedicated: false`, `broadcast: true`
- `interval` / `duration` control how often control data is interleaved with idle.
- Consider `disableNetworkGrant` / `convNetGrantDemand` depending on whether RF-only talkgroup
  steering is required.

```{note}
Voice-on-control (VOC) operation — using the control channel itself to also carry voice — is
**no longer recommended**, as most radios do not implement it correctly.
```

---

## Basic P25 trunking site (dedicated control & voice channels)

A dedicated control channel (CC) plus one or more voice channels (VCs). The CC directs radios
to VCs on demand.

### Requirements

- One modem/host per channel (one CC + N VCs), **or** a single host acting as CC with VC
  hosts connected via RPC.
- A correct [`iden_table.dat`](../software/dvmhost/dvmhost.md#the-identity-table) describing
  the channel plan (use the
  [`iden-channel-calculator`](https://github.com/DVMProject/iden-channel-calculator)).
- An [FNE](../software/dvmhost/dvmfne.md) if the site links to a wider network.

### Configuration

- **Control channel host:** `protocols.p25.control.enable: true`,
  `control.dedicated: true`; `system.config.channelId/channelNo` = the CC frequency.
- **Voice channel host(s):** `protocols.p25.control` disabled; each VC host's
  `system.config.channelId/channelNo` = its VC frequency.
- On the CC host, list every VC under `system.config.voiceChNo` with its `rpcAddress` /
  `rpcPort` / `rpcPassword` so the CC can grant and monitor them.
- On each VC host, set `system.config.controlCh` (`rpcAddress` / `rpcPort` / `rpcPassword`)
  back to the CC so it can report traffic status.
- Set matching `netId` (WACN), `sysId`, `rfssId`, `siteId` across all hosts at the site.

DMR (via TSCC) and NXDN trunking follow the same pattern using their respective `control`
sections.

---

## Multi-site / networked systems

To link multiple sites, conventional or trunked, each `dvmhost` connects to a
[`dvmfne`](../software/dvmhost/dvmfne.md) core:

- Set `network.enable: true` and point `network.address` / `port` / `password` at the FNE.
- Give each host a unique `network.id` (peer ID).
- The FNE's `talkgroup_rules.yml` defines which talkgroups route where.
- For trunked multi-site, configure `adj_site_map.yml` on the FNE so sites advertise each
  other as adjacent sites.
- Multiple FNEs can be peered together (`peers:` block) for very large networks; a spanning
  tree prevents routing loops.

Consoles ([Desktop Dispatch Console](../software/dvmconsole.md)) and analog bridges
([`dvmbridge`](../software/dvmhost/dvmbridge.md)) also connect to the FNE as peers.

## Source / further reading

- Annotated host config: <https://github.com/DVMProject/dvmhost/blob/master/configs/config.example.yml>
- Annotated FNE config: <https://github.com/DVMProject/dvmhost/blob/master/configs/fne-config.example.yml>
- Example `talkgroup_rules` / `adj_site_map` / ACL files: <https://github.com/DVMProject/dvmhost/tree/master/configs>
- Network stack technical note: `docs/TN.1000` in <https://github.com/DVMProject/dvmhost/tree/master/docs>
- `iden-channel-calculator` (channel plan / `iden_table.dat`): <https://github.com/DVMProject/iden-channel-calculator>
