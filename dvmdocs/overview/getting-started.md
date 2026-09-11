# Getting Started

## Decide what you want to do

It's important to know exactly what you're aiming to achieve with DVMProject so you can
determine which software and hardware components you'll need. Not sure? Take a look at the
[example system configurations](system-configs.md) for some typical setups.

Work through the questions below, then follow the links to the relevant guides.

### 1. What are you connecting to?

| You have… | Use… |
|-----------|------|
| A duplex analog repeater, or a separate Rx + Tx radio pair | [DVM-V1 duplex modem](../hardware/dvm-v1.md) running [`dvmfirmware`](../software/dvmfirmware/dvmfirmware.md) |
| A low-power bench/desktop setup, small coverage area | An MMDVM-style ADF7021 hotspot board running [`dvmfirmware-hs`](../software/dvmfirmware/dvmfirmware-hs.md) |
| A Motorola Quantar, AstroTAC comparator, or other V.24 P25 equipment | [DVM-V24 adapter](../hardware/dvm-v24.md) (P25 only, `dfsi` modem mode) |
| Only analog audio / an AllStar or USRP hub (no RF) | [`dvmbridge`](../software/dvmhost/dvmbridge.md) — no modem hardware required |

### 2. Conventional or trunked?

- **Conventional** — simplest. One channel, users pick talkgroups manually. Start here.
- **P25 conventional with control messaging** — adds affiliation and channel-grant behavior on
  a single conventional channel.
- **Trunked** — a dedicated control channel plus one or more voice channels. Requires an
  [`iden_table.dat`](../software/dvmhost/dvmhost.md) that correctly describes your channel plan.

### 3. Standalone or networked?

- **Standalone** — a single `dvmhost` with no network uplink. Fine for testing and isolated
  sites.
- **Networked** — one or more `dvmhost` instances connect to a
  [`dvmfne`](../software/dvmhost/dvmfne.md) (Fixed Network Equipment) core, which switches
  traffic between sites, consoles, and bridges. Multiple FNEs can be peered together for
  large networks.

## Next steps

1. Acquire and assemble your [hardware](../hardware/index.md).
2. [Build and install `dvmhost`](../software/dvmhost/installation.md) on a Linux host
   (Raspberry Pi 3+ or better, or an x86_64 machine).
3. Flash the appropriate [firmware](../software/dvmfirmware/index.md) to your modem or hotspot.
4. [Configure and calibrate `dvmhost`](../software/dvmhost/dvmhost.md).
5. If networking, [set up a `dvmfne`](../software/dvmhost/dvmfne.md).
6. Refer to the [example system configurations](system-configs.md) for wiring your specific
   system together.

## Source / further reading

- `dvmhost` README (build, calibration, CLI reference): <https://github.com/DVMProject/dvmhost/blob/master/README.md>
- Usage & support policy: <https://github.com/DVMProject/dvmhost/blob/master/usage_guidelines.md>
- All project repositories: [Introduction → Project repositories](introduction.md#project-repositories)
