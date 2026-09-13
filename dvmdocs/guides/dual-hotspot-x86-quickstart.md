# Quickstart: Dual-Hotspot P25 Trunked Node (x86)

```{warning}
**Work in progress.** This guide has not yet been run end-to-end or verified against real
hardware. The `dvmfirmware-hs` branch/target/binary-name details below have been checked
directly against the repository's `Makefile.STM32FX` on both `master` and `usb-support`; the
BOOT0 jumper location, exact `stm32flash` behavior, and the CC↔VC RPC wiring have not been
confirmed on physical boards. Treat it as a starting point, cross-check against your board's
own documentation and the
[P25 trunking reference](../overview/system-configs.md), and see something wrong? Fix it —
see [Contributing](../contributing.md).
```

This is an end-to-end path from a bare x86/x86_64 Linux box to a single machine running a
**minimal one-site P25 trunked system**: two **`MMDVM_HS_Dual`** duplex hotspot boards
connected over USB to one PC, one dedicated to the **control channel (CC)** and one dedicated
to a **voice channel (VC)**. It's a cheap way to stand up and learn real trunking — with an
actual duplex control channel, not a simplex approximation of one — without a repeater or a
second computer. One reasonably modern PC easily has the cores for two `dvmhost` instances.

```{note}
`MMDVM_HS_Dual` boards carry **two ADF7021 chips** — one dedicated to Rx, one to Tx — giving
true full-duplex operation, unlike single-chip simplex hotspots. That's exactly what a
dedicated control channel needs: it must keep transmitting control data continuously while
simultaneously receiving registration and grant requests. These boards are more commonly sold
as Raspberry Pi GPIO hats, but the same hardware/firmware target also runs over USB — see the
firmware note in step 3.
```

## The example system

| | Output (radio Rx) | Input (radio Tx) | Role |
|---|---|---|---|
| Hotspot 1 | **451.800 MHz** | **456.800 MHz** | Control Channel (CC) — dedicated, duplex |
| Hotspot 2 | **464.500 MHz** | **469.500 MHz** | Voice Channel (VC) — duplex |

Both pairs are a standard +5 MHz UHF duplex split.

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
- Two USB-connected `MMDVM_HS_Dual` boards, each flashed with the `usb-support` branch of
  [`dvmfirmware-hs`](../software/dvmfirmware/dvmfirmware-hs.md)
- A working one-site duplex P25 trunked system: radios register on the CC (451.800/456.800
  MHz) and get granted to the VC (464.500/469.500 MHz) for calls
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

:::{note}
**CH340/CH341 boards usually have no usable serial number.** The `grep -m1 serial` above walks
up the device tree and will happily match the first `ATTRS{serial}` it finds — but on CH340
clones (`idVendor=="1a86"`, `idProduct=="7523"`, driver `ch341-uart`) that's typically the
*PCI host controller's* serial (e.g. `0000:00:14.0`), not a per-board ID, and it comes back
identical for every board on the same controller. If both of your `grep` commands return the
same value, you have this case — the serial-based rule above won't work.

Key the rule on USB port path instead:

```bash
# get each board's port path
udevadm info -a -n /dev/ttyUSB0 | grep -m1 'KERNELS=="[0-9]'
udevadm info -a -n /dev/ttyUSB1 | grep -m1 'KERNELS=="[0-9]'
```

```{code-block} text
:caption: /etc/udev/rules.d/99-dvm-hotspots.rules

SUBSYSTEM=="tty", KERNELS=="1-1", SYMLINK+="dvm-cc"
SUBSYSTEM=="tty", KERNELS=="1-2", SYMLINK+="dvm-vc"
```

(Substitute the actual `KERNELS` values from your `udevadm` output — the port path, not
literally `1-1`/`1-2`.) Reload the same way. This ties the symlink to the physical USB port
rather than the board itself, so if you later swap which port a board is plugged into, the
symlinks swap with it — fine as long as you don't move the cables. `/dev/serial/by-path/` is
often already populated with equivalent stable names by systemd-udevd; check
`ls -la /dev/serial/by-path/` before writing a custom rule, since it may already give you what
you need.
:::

## 2. Build `dvmhost` once

One binary serves both instances.

```bash
git clone https://github.com/DVMProject/dvmhost.git
cd dvmhost && mkdir build && cd build
cmake ..
make
make strip
make tarball
sudo tar xvzf dvmhost_R04Gxx_<arch>.tar.gz -C /opt
```

```{note}
`make tarball` is the step that actually produces the `dvmhost_R04Gxx_<arch>.tar.gz` file —
it's easy to miss since `make strip` looks like the last build step. `<arch>` in the filename
isn't a placeholder to fill in; it's whatever CMake names it for your platform. Run
`ls *.tar.gz` in the build directory after `make tarball` to get the exact filename before
extracting.
```

See [Installation](../software/dvmhost/installation.md) for the full flag reference if you
need cross-compilation (not needed here — you're building natively for x86_64).

## 3. Flash both boards

```bash
git clone --recurse-submodules -b usb-support https://github.com/DVMProject/dvmfirmware-hs.git
cd dvmfirmware-hs
make -f Makefile.STM32FX mmdvm-hs-hat-dual
```

:::{note}
**Use the `usb-support` branch, not `master`, for USB-connected boards.** Confirmed directly
against both branches' `Makefile.STM32FX`: `master` only wires up GPIO/hat targets
(`mmdvm-hs-hat`, `mmdvm-hs-hat-dual`, `zumspot-adf7021`, `pihat-7021-r2`) under
`-DSTM32_USART1_HOST` — it references USB source files but never compiles them into any
target. `usb-support` carries the same board targets (minus `pihat-7021-r2`) plus
`lonestar-usb`, all rebuilt under `-DSTM32_USB_HOST` instead. That flag is the actual
USB-vs-GPIO switch, applied branch-wide to every target — it isn't tied to one board profile.

**The target is picked by your physical ADF7021 hardware, not by "USB vs GPIO."** For a
`MMDVM_HS_Dual`-style board — two ADF7021 chips, full duplex — the target is
`mmdvm-hs-hat-dual` whether you run it over its GPIO header or, as here, over USB. (For
reference: a single-ADF7021 board with no vendor-specific profile is `zumspot-adf7021`; a
single-ADF7021 `MMDVM_HS_Hat`-style board is `mmdvm-hs-hat`; a bare USB dongle with no
hat/GPIO form factor at all, e.g. a LoneStar USB stick, is `lonestar-usb`.)

Confirm against the actual target list on `usb-support` before building — it's a fast check
and cheaper than a wrong build:

```bash
grep -E '^[a-z0-9_-]+:' Makefile.STM32FX
```

**The output binary is also different on this branch.** Every target on `usb-support` routes
through a bootloader-enabled build (`bl` → `release_f1bl`), producing
`dvm-firmware-hs_f1bl.bin` — not `dvm-firmware-hs_f1.bin`, which is what `master`'s plain `hs`
build produces. Use the `_f1bl` filename below; confirm with `ls *.bin` in the build directory
if in doubt.
:::

Both boards are the same `MMDVM_HS_Dual` type, so build once and flash it to both:

1. Put the board in bootloader mode — jumper **BOOT0** (sometimes labeled **J1**) per your
   board's documentation; boards vary here, so check the seller's instructions.
2. Flash over its stable serial device:
   ```bash
   sudo stm32flash -v -w dvm-firmware-hs_f1bl.bin -R /dev/dvm-cc
   sudo stm32flash -v -w dvm-firmware-hs_f1bl.bin -R /dev/dvm-vc
   ```
3. Remove the boot jumper and power-cycle (or `-R` above already resets it).

## 4. Build the `iden_table.dat`

Both instances share **one row** of `iden_table.dat`. A row is a band anchor — a **Base Freq**
plus a **Spacing** (the channel raster) — not a slot for one specific frequency. You get an
actual operating frequency by picking a `channelNo` and computing
`Base Freq + (channelNo × Spacing)`; going the other direction,
`channelNo = (your frequency − Base Freq) / Spacing`. `Input Offset` is then added on top to
get the Tx/Rx split side (that's what makes these boards genuinely duplex, not simplex). See
the [Channel Identity Tables guide](iden-tables.md) for the general math.

```{note}
DVMProject currently supports FDMA modes only (P25 Phase 1, DMR, NXDN — no TDMA/Phase 2), and
the project's own example tables use a **6.25 kHz raster** even for 12.5 kHz-wide channels —
so `Spacing` below is `6.25`, not `12.5`. `Bandwidth` (the actual occupied channel width) stays
`12.5`. Getting `Spacing` wrong doesn't break any one frequency's math by itself, but it breaks
every *other* channel number on that identity row, and it doesn't match the raster every other
DVMProject table uses — don't deviate from it without a reason.
```

```{code-block} text
:caption: iden_table.dat

# ChId,Base Freq (Hz),Spacing (kHz),Input Offset (MHz),Bandwidth (kHz),
0,450000000,6.25,5.000,12.5,
```

One identity (`ChId 0`) anchored at 450.000000 MHz covers both channels — they're both on the
same +5 MHz duplex split, just at different points on the raster:

- **CC** (Hotspot 1) at 451.800 MHz: `channelNo = (451.800 − 450.000) MHz / 6.25 kHz = 1800 kHz / 6.25 kHz = 288`
  → `channelId: 0`, `channelNo: 288` → output 451.800000 MHz, input 456.800000 MHz
- **VC** (Hotspot 2) at 464.500 MHz: `channelNo = (464.500 − 450.000) MHz / 6.25 kHz = 14500 kHz / 6.25 kHz = 2320`
  → `channelId: 0`, `channelNo: 2320` → output 464.500000 MHz, input 469.500000 MHz

Create both instance directories and write this exact table — **not** the stock
`iden_table.example.dat`, which has different anchors — to the CC's directory, then copy it
to the VC as-is (both hosts read the same table):

```bash
sudo mkdir -p /opt/dvm/cc /opt/dvm/vc

sudo tee /opt/dvm/cc/iden_table.dat > /dev/null <<'EOF'
# ChId,Base Freq (Hz),Spacing (kHz),Input Offset (MHz),Bandwidth (kHz),
0,450000000,6.25,5.000,12.5,
EOF

sudo cp /opt/dvm/cc/iden_table.dat /opt/dvm/vc/iden_table.dat
```

## 5. Two configs, one trunked site

```bash
sudo cp /opt/dvm/configs/config.example.yml /opt/dvm/cc/config.yml
sudo cp /opt/dvm/configs/config.example.yml /opt/dvm/vc/config.yml
```

Both hosts run **P25 only**, full duplex, sharing the same site identity — only the channel
and CC/VC-specific fields differ.

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
  duplex: true
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
| `system.config.channelId` / `channelNo` | `0` / `288` | 451.800/456.800 MHz (step 4). |
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
      - channelId: 0
        channelNo: 2320
        rpcAddress: 127.0.0.1
        rpcPort: 9891          # the VC's own rpcPort, from its config below
        rpcPassword: "VC-RPC-PASSWORD"   # must match the VC's network.rpcPassword
```

### Hotspot 2 — Voice Channel (`/opt/dvm/vc/config.yml`)

| Field | Value | Why |
|-------|-------|-----|
| `system.modem.protocol.uart.port` | `/dev/dvm-vc` | Talks to the VC board. |
| `system.identity` / `cwId.callsign` | e.g. `N0CALL-VC` | Distinct CW/station ID. |
| `system.config.channelId` / `channelNo` | `0` / `2320` | 464.500/469.500 MHz (step 4). |
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
for the full field reference, and [system-configs](../overview/system-configs.md) for the
general pattern this quickstart is a worked example of.
```

Set `iAgreeNotToBeStupid: true` in both files.

## 6. Calibrate each independently

Run the [calibration procedure](../software/dvmhost/dvmhost.md#calibration) once per instance,
pointed at its own config. Since these are duplex boards, calibrate **both** Tx and Rx per
instance — there's no simplex shortcut here.

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
Description=DVM Host - P25 Control Channel (451.800/456.800 MHz)
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
  on 451.800 MHz — check its Tx LED and the CC's log for TSBK activity.
- A P25 radio programmed for WACN `DEE00`, System ID `123`, output 451.800 MHz / input
  456.800 MHz should show it's registered/affiliated on the system.
- Key up: the radio should be granted to 464.500/469.500 MHz for the call, and the VC's log
  should show the grant and the call.
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
