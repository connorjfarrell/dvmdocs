# Migrating from Pi-Star / MMDVMHost

If you already run a hotspot or repeater with Pi-Star, WPSD, or a hand-rolled MMDVMHost setup,
much of your knowledge transfers — but DVM is a different application with its own firmware,
config format, and networking model. This guide maps the concepts.

## What's the same

- The **hardware** — MMDVM-style STM32 modem and ADF7021 hotspot boards work with DVM.
- The **air interfaces** — P25, DMR, NXDN.
- The **calibration idea** — set levels for lowest BER.
- Raspberry Pi serial setup — disabling Bluetooth and the serial console on `/dev/ttyAMA0` is
  the same procedure.

## What's different

| Pi-Star / MMDVMHost | DVM |
|---------------------|-----|
| Stock MMDVM / MMDVM_HS firmware | **Required:** DVMProject firmware forks [`dvmfirmware`](../software/dvmfirmware/dvmfirmware.md) / [`dvmfirmware-hs`](../software/dvmfirmware/dvmfirmware-hs.md). Stock firmware will not work. |
| `MMDVM.ini` (INI format) | `config.yml` (YAML). No automatic conversion — rebuild it from [`config.example.yml`](https://github.com/DVMProject/dvmhost/blob/master/configs/config.example.yml). |
| Frequencies entered directly in the INI | Channel identity + channel number resolved via [`iden_table.dat`](iden-tables.md). |
| Reflectors / BrandMeister / P25 Reflector / XLX | A [`dvmfne`](../software/dvmhost/dvmfne.md) you run (or connect to), with `talkgroup_rules.yml` routing. Not compatible with BrandMeister/IPSC/reflector protocols. |
| Web dashboard bundled | Separate tools: `dvmmon` (TUI), `dvmhost_monitor`, `sysview`, the REST API + `pydvm`. |
| Nextion display support | Not a DVM focus. |
| Conventional hotspot only (mostly) | Full conventional **and** trunked (dedicated CC + VCs) operation. |

```{note}
DVM is not a drop-in replacement for a Pi-Star image and cannot connect to BrandMeister,
DMR+, or the common P25/NXDN reflector networks. It is its own network ecosystem built around
`dvmfne`. Decide whether that's what you want before migrating.
```

## Migration steps

1. **Back up** your working Pi-Star SD card.
2. Start from a clean Raspberry Pi OS (or Debian) install, or a dedicated Pi.
3. [Build and install `dvmhost`](../software/dvmhost/installation.md).
4. [Flash the DVM firmware](../software/dvmfirmware/index.md) for your exact board.
5. Note your frequencies, then build an [`iden_table.dat`](iden-tables.md) that produces them.
6. Write `config.yml` from the example — carry over your color code / NAC / RAN, callsign,
   and location.
7. [Calibrate](../software/dvmhost/dvmhost.md#calibration) from scratch — do not assume your
   Pi-Star RXOffset/TXOffset/levels transfer, though they're a useful starting point.
8. Point `network:` at your FNE, or run one locally.

## Source / further reading

- `dvmhost` README: <https://github.com/DVMProject/dvmhost/blob/master/README.md>
- Upstream MMDVM (shared lineage): <https://github.com/g4klx/MMDVM>
