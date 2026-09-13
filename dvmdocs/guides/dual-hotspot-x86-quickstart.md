# Quickstart: Dual-Hotspot Node on an x86 PC

```{warning}
**Work in progress.** This guide has not yet been run end-to-end or verified against a real
dual-board setup — in particular the USB bootloader-flashing steps (BOOT0 jumper location,
exact `stm32flash` invocation) are inferred from the same mechanism used elsewhere in these
docs, not confirmed on hardware. Treat it as a starting point, cross-check against your
board's own documentation, and see something wrong? Fix it — see [Contributing](../contributing.md).
```

This is an end-to-end path from a bare x86/x86_64 Linux box to a single machine running
**two independent hotspot channels** — e.g. one DMR hotspot and one P25 hotspot, or two
talkgroups/frequencies of the same mode — off of two separate USB hotspot boards. It's a
common alternative to running two Raspberry Pis: one reasonably modern PC easily has the
cores for two `dvmhost` instances.

```{note}
"Dual hotspot" here means **two separate boards, two separate `dvmhost` instances**. If you
instead have a single physical board with two ADF7021 chips on it for full-duplex operation
(e.g. an `MMDVM_HS_Dual_Hat`-style board), that's one hotspot with duplex Rx/Tx, normally
wired to a Raspberry Pi's GPIO rather than USB — see
[`dvmfirmware-hs`](../software/dvmfirmware/dvmfirmware-hs.md) instead. Everything below still
applies to *installing and configuring* it as a single instance.
```

## What you'll end up with

- One x86/x86_64 PC (Debian/Ubuntu-family recommended) running Linux
- A single `dvmhost` build, used by two systemd services with two config files
- Two USB-connected ADF7021 hotspot boards, each flashed with
  [`dvmfirmware-hs`](../software/dvmfirmware/dvmfirmware-hs.md)
- Two independently calibrated channels, standalone or both linked to an
  [FNE](../software/dvmhost/dvmfne.md)

## 1. Prep the PC

```bash
sudo apt-get update
sudo apt-get install build-essential cmake git libasio-dev libncurses-dev libssl-dev stm32flash
```

Plug in both hotspot boards. x86 doesn't have the Raspberry Pi GPIO/Bluetooth conflicts, so
there's no serial-console/Bluetooth prep step here — that section of the
[installation guide](../software/dvmhost/installation.md#raspberry-pi-preparation) is Pi-only
and does not apply.

### Get stable device names

With two identical USB-serial boards, `/dev/ttyUSB0` / `/dev/ttyUSB1` can swap between reboots
depending on enumeration order. Pin them with a udev rule keyed on each board's USB serial
number:

```bash
# find each board's serial number
udevadm info -a -n /dev/ttyUSB0 | grep -m1 serial
udevadm info -a -n /dev/ttyUSB1 | grep -m1 serial
```

```{code-block} text
:caption: /etc/udev/rules.d/99-dvm-hotspots.rules

SUBSYSTEM=="tty", ATTRS{idVendor}=="10c4", ATTRS{serial}=="AB0001", SYMLINK+="dvm-hs1"
SUBSYSTEM=="tty", ATTRS{idVendor}=="10c4", ATTRS{serial}=="AB0002", SYMLINK+="dvm-hs2"
```

(`idVendor` `10c4` is Silicon Labs CP210x, common on these boards; adjust for CH340/FTDI.)
Reload with `sudo udevadm control --reload && sudo udevadm trigger`, then confirm
`/dev/dvm-hs1` and `/dev/dvm-hs2` exist. Use these symlinks — not `/dev/ttyUSBx` — in both
config files below.

## 2. Build `dvmhost` once

One binary serves both instances.

```bash
git clone https://github.com/DVMProject/dvmhost.git
cd dvmhost && mkdir build && cd build
cmake ..
make
make strip
sudo tar xvzf dvmhost_R04Gxx_<arch>.tar.gz -C /opt
```

See [Installation](../software/dvmhost/installation.md) for the full flag reference if you
need cross-compilation (not needed here — you're building natively for x86_64).

## 3. Flash both boards

Build [`dvmfirmware-hs`](../software/dvmfirmware/dvmfirmware-hs.md) once, then flash each
board over its USB-serial port using the STM32 UART bootloader — the same mechanism used for
[DVM-V24](../hardware/dvm-v24.md), just via a plain USB-serial hotspot board instead of a
dedicated adapter:

```bash
git clone --recurse-submodules https://github.com/DVMProject/dvmfirmware-hs.git
cd dvmfirmware-hs
make -f Makefile.STM32FX <your-board-type>
```

1. Put the board in bootloader mode — jumper **BOOT0** (sometimes labeled **J1**) per your
   board's documentation; boards vary here, so check the seller's instructions.
2. Flash over its stable serial device:
   ```bash
   sudo stm32flash -v -w dvm-firmware-hs_f1.bin -R /dev/dvm-hs1
   sudo stm32flash -v -w dvm-firmware-hs_f1.bin -R /dev/dvm-hs2
   ```
3. Remove the boot jumper and power-cycle (or `-R` above already resets it).

## 4. Plan two channels in `iden_table.dat`

Both instances can share one `iden_table.dat` — just give each hotspot its own
`channelId`/`channelNo` row (or the same identity with two different channel numbers). See the
[Channel Identity Tables guide](iden-tables.md) if you need to build this from scratch. Example
covering a DMR hotspot and a P25 hotspot on different UHF frequencies:

```text
# ChId,Base Freq (Hz),Spacing (kHz),Input Offset (MHz),Bandwidth (kHz),
1,441000000,12.5,0,12.5,
```

- Hotspot 1 (DMR): `channelId: 1`, `channelNo: 0` → 441.000000 MHz
- Hotspot 2 (P25): `channelId: 1`, `channelNo: 8` → 441.100000 MHz

Copy this same `iden_table.dat` to both instances' config directories (or point both at one
shared path — either works since hotspots are simplex, single-channel).

## 5. Two configs, two identities

Lay out two independent instance directories:

```bash
sudo mkdir -p /opt/dvm/hotspot1 /opt/dvm/hotspot2
sudo cp /opt/dvm/configs/config.example.yml /opt/dvm/hotspot1/config.yml
sudo cp /opt/dvm/configs/config.example.yml /opt/dvm/hotspot2/config.yml
sudo cp /opt/dvm/configs/iden_table.example.dat /opt/dvm/hotspot1/iden_table.dat
sudo cp /opt/dvm/hotspot1/iden_table.dat /opt/dvm/hotspot2/iden_table.dat
```

Edit **each** `config.yml` — the fields that must differ between the two instances:

| Field | Hotspot 1 | Hotspot 2 | Why |
|-------|-----------|-----------|-----|
| `system.modem.protocol.uart.port` | `/dev/dvm-hs1` | `/dev/dvm-hs2` | Each talks to its own board. |
| `system.identity` / `cwId.callsign` | e.g. `N0CALL-1` | `N0CALL-2` | Distinct CW/station ID. |
| `system.config.channelId` / `channelNo` | `1` / `0` | `1` / `8` | From step 4. |
| `protocols.dmr.enable` / `protocols.p25.enable` / etc. | pick one mode (or mixed) | pick one mode (or mixed) | Independent per hotspot. |
| `system.config.colorCode` (DMR) or `nac` (P25) or `ran` (NXDN) | your values | your values | Must not collide if the two are ever audible to the same radio. |
| `network.id` (if networked) | unique peer ID | unique peer ID | FNE requires unique peer IDs. |
| `network.rpcAddress` / `rpcPort`, `network.restPort` | e.g. `9890` / `9990` | e.g. `9891` / `9991` | Two processes can't bind the same port on one host. |
| `log.filePath` / `log.fileRoot` | e.g. `/opt/dvm/hotspot1`, `DVM1` | e.g. `/opt/dvm/hotspot2`, `DVM2` | Keep logs from overwriting each other. |

Set `iAgreeNotToBeStupid: true` in both.

## 6. Calibrate each independently

Run the [calibration procedure](../software/dvmhost/dvmhost.md#calibration) once per instance,
pointed at its own config:

```bash
/opt/dvm/bin/dvmhost -c /opt/dvm/hotspot1/config.yml --setup
/opt/dvm/bin/dvmhost -c /opt/dvm/hotspot2/config.yml --setup
```

Do this one at a time — running both boards' Tx simultaneously while you're trying to read a
radio's BER on one of them will just confuse you.

## 7. Two systemd services

```{code-block} ini
:caption: /etc/systemd/system/dvmhost-hotspot1.service

[Unit]
Description=DVM Host - Hotspot 1
After=network.target

[Service]
ExecStart=/opt/dvm/bin/dvmhost -f -c /opt/dvm/hotspot1/config.yml
Restart=on-failure
User=dvm

[Install]
WantedBy=multi-user.target
```

Duplicate as `dvmhost-hotspot2.service` pointing at `hotspot2/config.yml`. Then:

```bash
sudo useradd -r -s /usr/sbin/nologin dvm   # if you don't already run dvmhost as its own user
sudo usermod -aG dialout dvm               # serial port access
sudo systemctl daemon-reload
sudo systemctl enable --now dvmhost-hotspot1 dvmhost-hotspot2
journalctl -u dvmhost-hotspot1 -f          # watch one instance's log
```

## 8. Standalone, or networked

- **Standalone** — leave `network.enable: false` on both. Each hotspot works independently;
  nothing more to do.
- **Networked** — point both at the same or different [FNE](../software/dvmhost/dvmfne.md)
  with their (already-unique) `network.id` peer IDs. This is also how you'd bridge the two
  modes together operationally — e.g. patch a DMR talkgroup to a P25 talkgroup via
  `talkgroup_rules.yml` rewrites, or [`dvmpatch`](../software/dvmhost/supplementary.md).

## Verify

- `systemctl status dvmhost-hotspot1 dvmhost-hotspot2` — both `active (running)`.
- Each board's LED behaves as expected on Tx/Rx.
- Bring up a radio on each channel and confirm two-way digital audio.
- If networked, confirm both peers show connected on the FNE
  ([`sysview`](../software/dvmhost/supplementary.md) or the REST API).

## Source / further reading

- [Installation](../software/dvmhost/installation.md), [`dvmhost`](../software/dvmhost/dvmhost.md) configuration & calibration
- [`dvmfirmware-hs`](../software/dvmfirmware/dvmfirmware-hs.md)
- [Channel Identity Tables](iden-tables.md)
- [FNE Networking & Topology](networking.md)
- [Troubleshooting](troubleshooting.md) if a board won't enumerate or calibration won't settle
