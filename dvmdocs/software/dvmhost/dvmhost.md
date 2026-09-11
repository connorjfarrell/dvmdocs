# `dvmhost` Digital Radio Host Application

`dvmhost` is the primary data-processing application. It connects to a DVM modem — either the
**air interface** (repeater or hotspot) or the **P25 V.24 / DFSI** interface to commercial
equipment — and implements the DMR, P25, and/or NXDN protocol state machines for a single
channel.

```{warning}
`dvmhost` is for amateur and educational use only and must never be used in public safety or
life safety critical applications. See the [Introduction](../../overview/introduction.md).
```

## Configuration overview

`dvmhost` is configured with a single YAML file, conventionally `config.yml`. The annotated
[`config.example.yml`](https://github.com/DVMProject/dvmhost/blob/master/configs/config.example.yml)
in the repository documents every option and is the authoritative reference — this page only
orients you to the major sections.

| Section | What it controls |
|---------|------------------|
| `log` | Log levels, file paths, syslog. |
| `network` | FNE connection (`enable`, `id`, `address`, `port`, `password`), optional link encryption, lookup-table updates, and the **RPC** and **REST API** listeners. |
| `protocols.dmr` / `.p25` / `.nxdn` | Per-mode enable flag, control-channel (CC / TSCC) settings, hang timers, BER silence thresholds, queue sizes, verbose/debug. |
| `system` | `identity`, call `timeout`, `duplex`, `fixedMode` / `modeHang`, location `info`, and the `config` sub-section. |
| `system.config` | `authoritative` / `supervisor` flags, `channelId` / `channelNo`, `voiceChNo` list, `controlCh` RPC back-link, `colorCode` / `nac` / `ran`, and network identifiers (`dmrNetId`, `netId` (WACN), `sysId`, `rfssId`, `siteId`). |
| `system.cwId` | Morse station ID. |
| `system.modem` | Modem port (`protocol.type`: `null` / `uart`; `protocol.mode`: `air` / `dfsi`), Rx/Tx invert, DC blocker, FIFO lengths, hotspot ADF7021 tuning (`hotspot:`), repeater symbol-level trims (`repeater:`), softpot values, and V.24 (`dfsi:`) parameters. |
| `system.iden_table` | Path to `iden_table.dat` and its refresh interval. |
| `system.radio_id` / `system.talkgroup_id` | Paths to the RID ACL (`rid_acl.dat`) and talkgroup rules (`talkgroup_rules.yml`), refresh intervals, and whether ACLs are enforced. |

```{note}
Set `iAgreeNotToBeStupid: true` to acknowledge the licence and usage restrictions — `dvmhost`
will not run otherwise.
```

### The identity table

`iden_table.dat` describes your channel plan. `dvmhost` uses it to convert a
`channelId` + `channelNo` pair into an actual Rx/Tx frequency. Those calculations drive:

- the operating frequency of a hotspot,
- the Rx/Tx split of a repeater/modem,
- the over-the-air messages that tell trunking radios which frequency to move to.

Getting this right is the **first** configuration step for any trunked system. The
[`iden-channel-calculator`](https://github.com/DVMProject/iden-channel-calculator) CLI tool
generates entries and maps frequencies to channel numbers.

## Running `dvmhost`

```
usage: ./dvmhost [-vhdf] [--syslog] [--setup] [--cal] [--boot]
                 [-c <configuration file>] [--remote [-a <address>] [-p <port>]]

  -v        show version information
  -h        show help
  -d        force modem debug
  -f        foreground mode (do not daemonize)
  --syslog  force logging to syslog
  --setup   TUI setup and calibration mode
  --cal     simple (CLI) calibration mode
  --boot    connect to the modem and reboot it into bootloader mode
  -c <file> configuration file to use
  --remote  remote modem mode (-a address, -p port, -P local listen port)
```

Normal operation:

```bash
/opt/dvm/bin/dvmhost -c /opt/dvm/config.yml
```

Or via the installed systemd service (`systemctl start dvmhost`).

## Initial setup

The setup TUI (`--setup`) is available when the host was built with `ENABLE_SETUP_TUI`. It can
be done manually by editing `config.yml` and `iden_table.dat` instead.

1. In `config.yml` → `system.modem.protocol`, set `uart.port` and `uart.speed` (defaults
   `/dev/ttyUSB0`, `115200`), and set `protocol.mode` to `air` (modem/hotspot) or `dfsi`
   (DVM-V24 — firmware ≥ 2.0 required).
2. Start `dvmhost -c config.yml --setup`.
3. Work through the **Setup** menu:
   - *Logging & Data Configuration* — log paths/levels and data-file paths.
   - *System Configuration* — modem port/speed, system and mode settings.
   - *Site Parameters* — CW ID and site parameters.
   - *Channel Configuration* — the channel for this modem.
4. **File → Save Settings**, then **File → Quit** (some changes need a restart).

## Calibration

Calibration applies to the **air interface** modem/hotspot only. The best tool is a radio that
can transmit and measure BER (e.g. an XTS with ASTRO25 Tuner "Bit Error Rate" / "Transmitter
Test Pattern" functions), or an RF service monitor.

### Air-interface transmit calibration

1. `dvmhost -c config.yml --setup` (or `--cal` for the CLI version).
2. **Calibrate → Operational Mode →** pick a Tx test pattern, e.g. *[Tx] DMR BS 1031 Hz Test
   Pattern* or *[Tx] P25 1011 Hz Test Pattern (NAC293 ID1 TG1)*.
3. Open **Level Adjustment** (F5). Set **TX Level = 50**.
4. Set the hardware Tx potentiometer (if any) to minimum.
5. Start Tx (F12). Adjust the hardware pot for lowest BER, then fine-tune with the software TX
   Level. Stop Tx.
6. **File → Save Settings**, then Quit.

In `--cal` mode the keys are: `` ` `` show values, `M`/`P` DMR/P25 BS pattern, spacebar toggle
Tx, `T`/`t` raise/lower TXLevel, `s` save, `q` quit.

### Air-interface receive calibration

Same as above but pick an **[Rx]** test pattern, set **RX Level = 50**, minimise the hardware
Rx pot(s), and watch the Receive BER window (top-right of the setup TUI) while adjusting for
the lowest BER. In `--cal` mode: `M`/`P` for the BS pattern, then `J`/`j` for the MS pattern;
`R`/`r` adjust RXLevel.

### Calibration notes

- Target overall **analog FM deviation of 2.75–2.83 kHz** (aim for 2.80 kHz). You may need to
  "de-tune" a commercial radio whose alignment limits deviation to 2.5 kHz for 12.5 kHz
  channels.
- BER consistently **> 10 %** usually means the radio/hotspot is off frequency (keep reference
  drift within ±150 Hz), a DC offset in the signal path, or wrong deviation levels.
- For hotspots, you may need to toggle **AFC** or change the **gain mode** (`system.modem.
  hotspot.adfGainMode`, `afcEnable`). In trunking mode, a 90° antenna adapter plus "Low" gain
  can prevent Rx desense.
- Only adjust symbol levels (`system.modem.repeater.*SymLvl*Adj`) with proper RF test
  equipment.

### Hotspot calibration with a service monitor

1. Zero all frequency offsets; Tx Deviation nominal (50).
2. Spectrum-analyzer mode + *P25 1200 Hz Tone* (`z`): adjust Tx deviation to null the centre
   carrier while keeping the side lobes and a clean 1200 Hz tone.
3. FM-deviation mode + *P25 1011 Hz Test Pattern* (`P`): adjust Tx deviation toward ~2.83 kHz
   average.
4. Frequency-error mode + *P25 1011 Hz Test Pattern*: note the average error and apply the
   opposite Tx Frequency Adjustment (e.g. +200 Hz error → enter −200 Hz). The Rx Frequency
   Adjustment normally follows the Tx value.

Without a service monitor, skip step 2 and use a BER-capable radio, varying deviation and
frequency offset together to minimise BER.

## Source / further reading

- README (build, calibration, CLI): <https://github.com/DVMProject/dvmhost/blob/master/README.md>
- Annotated host config: [`config.example.yml`](https://github.com/DVMProject/dvmhost/blob/master/configs/config.example.yml)
- Air interface tuning: `docs/TN.1101 - dvmhost Air Interface Tuning.adoc`
- Host REST API: `docs/TN.1101 - DVMHost REST API Documentation.md`
- Modem protocol: `docs/TN.1001 - Modem Protocol.adoc`
