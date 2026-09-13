# Quickstart: Dual-Hotspot P25 Trunked Node (x86)

```{warning}
**Work in progress.** This guide has not yet been run end-to-end or verified against real
hardware — in particular the USB bootloader-flashing steps (BOOT0 jumper location, exact
`stm32flash` invocation) are inferred from the same mechanism used elsewhere in these docs,
not confirmed on hardware, and the trunking RPC wiring has not been tested on an actual
simplex-hotspot pair. Treat it as a starting point, cross-check against your board's own
documentation and the [P25 trunking reference](../overview/system-configs.md),
and see something wrong? Fix it — see [Contributing](../contributing.md).
```

This is an end-to-end path from a bare x86/x86_64 Linux box to a single machine running a
**minimal one-site P25 trunked system**: two USB hotspot boards on one PC, one dedicated to
the **control channel (CC)** and one dedicated to a **voice channel (VC)**. It's a cheap way to
stand up and learn trunking without a repeater or a second computer — one reasonably modern PC
easily has the cores for two `dvmhost` instances.

```{note}
"Dual hotspot" here means **two separate boards, two separate `dvmhost` instances**, one CC and
one VC, coordinating over a local RPC link. If you instead have a single physical board with
two ADF7021 chips on it for full-duplex operation (e.g. an `MMDVM_HS_Dual_Hat`-style board),
that's one *duplex* hotspot, normally wired to a Raspberry Pi's GPIO rather than USB — see
[`dvmfirmware-hs`](../software/dvmfirmware/dvmfirmware-hs.md) instead.

Hotspot boards are typically low-power **simplex** boards (one frequency, shared Tx/Rx). A
real trunking site normally wants duplex (repeater) hardware so the CC can transmit and
receive at the same time. This guide's CC and VC each sit on their own simplex frequency
instead, which is a fine way to *learn and test* trunking behavior, but treat it as a lab
setup, not a production site.
```

## The example system

| | Frequency | Role |
|---|---|---|
| Hotspot 1 | **451.800 MHz** | Control Channel (CC) — dedicated |
| Hotspot 2 | **464.500 MHz** | Voice Channel (VC) |

| Parameter | Value |
|---|---|
| WACN (`netId`) | `DEE00` |
| System ID (`sysId`) | `123` |
| RFSS ID | `1` |
| Site ID | `1` |
| NAC | `293` *(example — pick your own, must match on both channels)* |

## What you'll end up with

- One x86/x86_64 PC (Debian/Ubuntu-family recommended) running Linux
- A single `dvmhost` build, used by two systemd services with two config files
- Two USB-connected ADF7021 hotspot boards, each flashed with
  [`dvmfirmware-hs`](../software/dvmfirmware/dvmfirmware-hs.md)
- A working one-site P25 trunked system: radios register on the CC (451.800 MHz) and get
  granted to the VC (464.500 MHz) for calls
- Standalone, or linked to an [FNE](../software/dvmhost/dvmfne.md) for a wider network

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

SUBSYSTEM=="tty", ATTRS{idVendor}=="10c4", ATTRS{serial}=="AB0001", SYMLINK+="dvm-cc"
SUBSYSTEM=="tty", ATTRS{idVendor}=="10c4", ATTRS{serial}=="AB0002", SYMLINK+="dvm-vc"
```

(`idVendor` `10c4` is Silicon Labs CP210x, common on these boards; adjust for CH340/FTDI.)
Reload with `sudo udevadm control --reload && sudo udevadm trigger`, then confirm
`/dev/dvm-cc` and `/dev/dvm-vc` exist. Use these symlinks — not `/dev/ttyUSBx` — in both
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
   sudo stm32flash -v -w dvm-firmware-hs_f1.bin -R /dev/dvm-cc
   sudo stm32flash -v -w dvm-firmware-hs_f1.bin -R /dev/dvm-vc
   ```
3. Remove the boot jumper and power-cycle (or `-R` above already resets it).

## 4. Build the `iden_table.dat`

Both instances share one `iden_table.dat` — a row per channel, each anchored directly on its
operating frequency (no repeater offset needed since these are simplex boards). See the
[Channel Identity Tables guide](iden-tables.md) for the general math.

```{code-block} text
:caption: iden_table.dat

# ChId,Base Freq (Hz),Spacing (kHz),Input Offset (MHz),Bandwidth (kHz),
0,451800000,12.5,0.000,12.5,
1,464500000,12.5,0.000,12.5,
```

- **CC** (Hotspot 1): `channelId: 0`, `channelNo: 0` → 451.800000 MHz
- **VC** (Hotspot 2): `channelId: 1`, `channelNo: 0` → 464.500000 MHz

Copy the same `iden_table.dat` to both instances' config directories.

## 5. Two configs, one trunked site

Lay out two independent instance directories:

```bash
sudo mkdir -p /opt/dvm/cc /opt/dvm/vc
sudo cp /opt/dvm/configs/config.example.yml /opt/dvm/cc/config.yml
sudo cp /opt/dvm/configs/config.example.yml /opt/dvm/vc/config.yml
sudo cp /opt/dvm/configs/iden_table.example.dat /opt/dvm/cc/iden_table.dat
sudo cp /opt/dvm/cc/iden_table.dat /opt/dvm/vc/iden_table.dat
```

Both hosts run **P25 only**, simplex, sharing the same site identity — only the channel and
CC/VC-specific fields differ.

### Common to both `config.yml`

```yaml
protocols:
  p25:
    enable: true
  dmr:
    enable: false
  nxdn:
    enable: false

system:
  duplex: false
  simplexSameFrequency: true
  config:
    nac: 293          # example — pick your own; must match on both channels
    netId: DEE00       # WACN
    sysId: 123
    rfssId: 1
    siteId: 1
  iden_table:
    file: iden_table.dat
```

### Hotspot 1 — Control Channel (`/opt/dvm/cc/config.yml`)

| Field | Value | Why |
|-------|-------|-----|
| `system.modem.protocol.uart.port` | `/dev/dvm-cc` | Talks to the CC board. |
| `system.identity` / `cwId.callsign` | e.g. `N0CALL-CC` | Distinct CW/station ID. |
| `system.config.channelId` / `channelNo` | `0` / `0` | 451.800 MHz (step 4). |
| `system.config.authoritative` | `true` | This host handles channel grants. |
| `protocols.p25.control.enable` | `true` | Turns this channel into a CC. |
| `protocols.p25.control.dedicated` | `true` | Dedicated CC — signalling only, no voice. |
| `protocols.p25.control.broadcast` | `true` | Actually transmit CC data. |
| `network.rpcAddress` / `rpcPort` / `rpcPassword` | `127.0.0.1` / `9890` / *(pick a password)* | This host's own RPC listener — the VC will call into it. |
| `system.config.voiceChNo` | one entry, see below | Tells the CC about its VC. |
| `log.filePath` / `log.fileRoot` | e.g. `/opt/dvm/cc`, `DVMCC` | Keep logs separate. |

```yaml
system:
  config:
    voiceChNo:
      - channelId: 1
        channelNo: 0
        rpcAddress: 127.0.0.1
        rpcPort: 9891          # the VC's own rpcPort, from its config below
        rpcPassword: "VC-RPC-PASSWORD"   # must match the VC's network.rpcPassword
```

### Hotspot 2 — Voice Channel (`/opt/dvm/vc/config.yml`)

| Field | Value | Why |
|-------|-------|-----|
| `system.modem.protocol.uart.port` | `/dev/dvm-vc` | Talks to the VC board. |
| `system.identity` / `cwId.callsign` | e.g. `N0CALL-VC` | Distinct CW/station ID. |
| `system.config.channelId` / `channelNo` | `1` / `0` | 464.500 MHz (step 4). |
| `system.config.authoritative` | `false` | The CC handles grants, not this host. |
| `protocols.p25.control.enable` | `false` | This channel only carries voice. |
| `network.rpcAddress` / `rpcPort` / `rpcPassword` | `127.0.0.1` / `9891` / `"VC-RPC-PASSWORD"` | This host's own RPC listener — referenced by the CC's `voiceChNo` entry above. |
| `system.config.controlCh` | see below | Points this VC back at the CC. |
| `log.filePath` / `log.fileRoot` | e.g. `/opt/dvm/vc`, `DVMVC` | Keep logs separate. |

```yaml
system:
  config:
    controlCh:
      rpcAddress: 127.0.0.1
      rpcPort: 9890            # the CC's own rpcPort, from its config above
      rpcPassword: "ULTRA-VERY-SECURE-DEFAULT"   # must match the CC's network.rpcPassword
      notifyEnable: true
      presence: 120
```

```{note}
The two `rpcPassword` pairs above are **local loopback secrets between your own two
processes**, not something radios or a network ever see — pick anything, just make sure each
side's value matches what the other side expects. This is the same CC↔VC RPC pattern used by
[`config.example.yml`](https://github.com/DVMProject/dvmhost/blob/master/configs/config.example.yml)
for a multi-host trunking site; see [`dvmhost`](../software/dvmhost/dvmhost.md#configuration-overview)
for the full field reference, and
[system-configs](../overview/system-configs.md)
for the general pattern this quickstart is a worked example of.
```

Set `iAgreeNotToBeStupid: true` in both files.

## 6. Calibrate each independently

Run the [calibration procedure](../software/dvmhost/dvmhost.md#calibration) once per instance,
pointed at its own config:

```bash
/opt/dvm/bin/dvmhost -c /opt/dvm/cc/config.yml --setup
/opt/dvm/bin/dvmhost -c /opt/dvm/vc/config.yml --setup
```

Do this one at a time — running both boards' Tx simultaneously while you're trying to read a
radio's BER on one of them will just confuse you. Calibrate the CC first, since you need it
transmitting before a radio will show you anything on the VC at all.

## 7. Two systemd services

```{code-block} ini
:caption: /etc/systemd/system/dvmhost-cc.service

[Unit]
Description=DVM Host - P25 Control Channel (451.800 MHz)
After=network.target

[Service]
ExecStart=/opt/dvm/bin/dvmhost -f -c /opt/dvm/cc/config.yml
Restart=on-failure
User=dvm

[Install]
WantedBy=multi-user.target
```

Duplicate as `dvmhost-vc.service` pointing at `/opt/dvm/vc/config.yml`. Start the **CC
first**, then the VC:

```bash
sudo useradd -r -s /usr/sbin/nologin dvm   # if you don't already run dvmhost as its own user
sudo usermod -aG dialout dvm               # serial port access
sudo systemctl daemon-reload
sudo systemctl enable --now dvmhost-cc
sudo systemctl enable --now dvmhost-vc
journalctl -u dvmhost-cc -f                # watch the CC's log
```

## 8. Standalone, or networked

- **Standalone** — leave `network.enable: false` on both. This one site works on its own;
  nothing more to do.
- **Networked** — point both at an [FNE](../software/dvmhost/dvmfne.md) with unique
  `network.id` peer IDs, so this site can be one of several sites in a wider trunked network
  (adjacent-site broadcasts, roaming, etc. — see [FNE Networking](networking.md)).

## Verify

- `systemctl status dvmhost-cc dvmhost-vc` — both `active (running)`.
- The CC board should be transmitting continuously (or on its configured broadcast interval)
  — check its Tx LED and the CC's log for TSBK activity.
- A P25 radio programmed for WACN `DEE00`, System ID `123`, and the CC frequency (451.800 MHz)
  should show it's registered/affiliated on the system.
- Key up: the radio should be granted to 464.500 MHz for the call, and the VC's log should
  show the grant and the call.
- If networked, confirm both peers show connected on the FNE
  ([`sysview`](../software/dvmhost/supplementary.md) or the REST API).

## Source / further reading

- [Installation](../software/dvmhost/installation.md), [`dvmhost`](../software/dvmhost/dvmhost.md) configuration & calibration
- [Basic P25 trunking site](../overview/system-configs.md) — the general pattern
- [`dvmfirmware-hs`](../software/dvmfirmware/dvmfirmware-hs.md)
- [Channel Identity Tables](iden-tables.md)
- [FNE Networking & Topology](networking.md)
- [Troubleshooting](troubleshooting.md) if a board won't enumerate, calibration won't settle,
  or grants aren't happening
