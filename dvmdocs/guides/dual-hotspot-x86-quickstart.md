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

### Device names

This guide uses `/dev/ttyUSB0` for the CC board and `/dev/ttyUSB1` for the VC board directly,
with no udev symlinks. That's simpler, but it means **you** are responsible for keeping the
mapping stable instead of udev:

- **Label both boards and both USB ports/cables** physically (a piece of tape saying "CC" /
  "VC" on the board and the port it plugs into) before you do anything else. Always plug each
  board into its labeled port.
- Linux numbers `/dev/ttyUSB*` in **enumeration order**, not by port location — if you ever
  plug them in one at a time, unplug one, or add a third USB-serial device, the numbering can
  shift. After (re)booting with both boards connected, confirm which is which before trusting
  `ttyUSB0`/`ttyUSB1`:
  ```bash
  ls -la /dev/serial/by-path/    # often already stable per physical port, worth checking first
  dmesg | grep -i ttyUSB         # shows the order they were detected in, most recent last
  ```
  If it ever looks swapped, either fix the physical cabling to match the labels, or swap the
  `uart.port` values in the two `config.yml` files below to match reality — just keep the
  labels and the configs in agreement.
- If you'd rather have Linux enforce this instead of doing it by hand (recommended once you've
  moved past initial bring-up), key a udev rule off each board's USB serial number or port
  path — see [Installation](../software/dvmhost/installation.md) for the general approach; the
  DVM-V24 guide's [firmware flashing section](../hardware/dvm-v24.md) shows the same pattern
  for a single board.

:::{note}
Whatever the device path, add `ENV{ID_MM_DEVICE_IGNORE}="1"` for these boards' vendor/product
IDs so **ModemManager** — installed by default on most desktop Ubuntu/Debian systems — doesn't
probe them. ModemManager opening/closing the port to check whether it's a cellular modem is
one of the most common causes of "cannot communicate with modem" even when the device path is
correct. A minimal rule that just does that, with no renaming:

```{code-block} text
:caption: /etc/udev/rules.d/99-dvm-hotspots.rules

SUBSYSTEM=="tty", ATTRS{idVendor}=="10c4", ENV{ID_MM_DEVICE_IGNORE}="1"
```

(`10c4` is Silicon Labs CP210x, common on these boards; add another line with the right
`idVendor` if yours use CH340 (`1a86`) or FTDI (`0403`) instead.) Reload with
`sudo udevadm control --reload && sudo udevadm trigger`. See the troubleshooting note after
step 3 if you're still stuck.
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

Both boards are the same `MMDVM_HS_Dual` type, so build once and flash it to both. Confirm
which board is on which labeled port (per the [Device names](#device-names) note above) before
you start — flashing the "wrong" board isn't harmful, but it'll make you doubt your labeling
later if you don't check now:

1. Put the board in bootloader mode — jumper **BOOT0** (sometimes labeled **J1**) per your
   board's documentation; boards vary here, so check the seller's instructions.
2. Flash over its device:
   ```bash
   sudo stm32flash -v -w dvm-firmware-hs_f1bl.bin -R /dev/ttyUSB0   # CC board
   sudo stm32flash -v -w dvm-firmware-hs_f1bl.bin -R /dev/ttyUSB1   # VC board
   ```
3. Remove the boot jumper and power-cycle (or `-R` above already resets it).

:::{note}
**"Could not communicate with modem" even though the device path is correct?** This is
`dvmhost` failing to talk to the *firmware*, not a path problem — check, in order:

1. **BOOT0 jumper still fitted.** If it's still bridged after flashing, the board stays in the
   ST bootloader (which speaks the `stm32flash` protocol, not the DVM modem protocol) and will
   never respond to `dvmhost`. Remove it and power-cycle the board.
2. **ModemManager or `brltty` has the port open.** Both are common on desktop Ubuntu/Debian
   installs and grab newly-plugged USB-serial devices to probe them. The
   `ENV{ID_MM_DEVICE_IGNORE}="1"` udev rule above handles ModemManager for these two devices
   specifically; if `brltty` is installed and you don't need it,
   `sudo systemctl disable --now brltty.service` (or `apt remove brltty`) stops it grabbing
   ports too. `sudo lsof /dev/ttyUSB0` (or `fuser`) shows you if something else already has the
   port open.
3. **`dialout` group membership hasn't taken effect yet.** `usermod -aG` only applies to *new*
   login sessions — if you're testing interactively as a user you just added, log out and back
   in (or `newgrp dialout` in the current shell) before assuming permissions are the problem.
4. **Wrong baud rate.** `system.modem.protocol.uart.speed` must be `115200` unless you changed
   it in the firmware — the configs in step 5 already use the default.
5. **Board never actually flashed.** Confirm `stm32flash` printed `Wrote and verified` with no
   errors for *each* board, not just the first — a failed second flash is an easy thing to miss
   when running both commands back-to-back.
:::

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
4,450000000,6.25,5.000,12.5,
```

```{note}
The identity number (`4` here) isn't arbitrary — it must match whatever identity slot **your
radio's own codeplug/bandplan** expects for this band segment. `dvmhost` doesn't care what
number you pick as long as `channelId` in `config.yml` matches the row, but the *radio* only
knows how to resolve OTA channel grants against the identity numbers it was programmed with.
If your bandplan uses identity 4 for this band, the table row and both configs' `channelId`
must all say `4`, not `0`.
```

One identity (`ChId 4`) anchored at 450.000000 MHz covers both channels — they're both on the
same +5 MHz duplex split, just at different points on the raster:

- **CC** (Hotspot 1) at 451.800 MHz: `channelNo = (451.800 − 450.000) MHz / 6.25 kHz = 1800 kHz / 6.25 kHz = 288`
  → `channelId: 4`, `channelNo: 288` → output 451.800000 MHz, input 456.800000 MHz
- **VC** (Hotspot 2) at 464.500 MHz: `channelNo = (464.500 − 450.000) MHz / 6.25 kHz = 14500 kHz / 6.25 kHz = 2320`
  → `channelId: 4`, `channelNo: 2320` → output 464.500000 MHz, input 469.500000 MHz

Create both instance directories and write this exact table — **not** the stock
`iden_table.example.dat`, which has different anchors — to the CC's directory, then copy it
to the VC as-is (both hosts read the same table):

```bash
sudo mkdir -p /opt/dvm/cc /opt/dvm/vc

sudo tee /opt/dvm/cc/iden_table.dat > /dev/null <<'EOF'
# ChId,Base Freq (Hz),Spacing (kHz),Input Offset (MHz),Bandwidth (kHz),
4,450000000,6.25,5.000,12.5,
EOF

sudo cp /opt/dvm/cc/iden_table.dat /opt/dvm/vc/iden_table.dat
```

## 5. Two configs, one trunked site

Both hosts run **P25 only**, full duplex, sharing the same site identity — the two configs
below are [`config.example.yml`](https://github.com/DVMProject/dvmhost/blob/master/configs/config.example.yml)
with comments stripped, values set for this site, and the CC/VC-specific fields (in
**bold** below) differing between them. Write each one directly:

```{note}
The two `rpcPassword` values (`CC-RPC-PASSWORD` / `VC-RPC-PASSWORD`) are **local loopback
secrets between your own two processes**, not something radios or a network ever see — change
them to anything, just keep each side's value matching what the other side expects (the CC's
`voiceChNo[].rpcPassword` must equal the VC's `network.rpcPassword`, and the VC's
`controlCh.rpcPassword` must equal the CC's `network.rpcPassword`). This is the same CC↔VC RPC
pattern documented in [`dvmhost`](../software/dvmhost/dvmhost.md#configuration-overview) and
[system-configs](../overview/system-configs.md).
```

### Hotspot 1 — Control Channel (`/opt/dvm/cc/config.yml`)

Differs from the VC below in: `log` paths, `network.rpcPort`/`rpcPassword`,
**`protocols.p25.control`** (this is what makes it a CC), **`system.config.channelNo` (288)**,
**`authoritative: true`**, the **`voiceChNo`** entry pointing at the VC, `system.identity` /
`cwId.callsign`, and `modem.protocol.uart.port`.

```bash
sudo tee /opt/dvm/cc/config.yml > /dev/null <<'EOF'
daemon: true
iAgreeNotToBeStupid: true
log:
    disableNonAuthoritiveLogging: false
    displayLevel: 1
    fileLevel: 1
    useSyslog: false
    filePath: /opt/dvm/cc
    activityFilePath: /opt/dvm/cc
    fileRoot: DVMCC
network:
    enable: false
    id: 100001
    address: 127.0.0.1
    port: 62031
    password: "PASSWORD"
    encrypted: false
    presharedKey: "000102030405060708090A0B0C0D0E0F000102030405060708090A0B0C0D0E0F"
    jitter: 360
    slot1: true
    slot2: true
    updateLookups: false
    saveLookups: false
    allowActivityTransfer: true
    allowDiagnosticTransfer: true
    allowStatusTransfer: true
    packetDump: false
    debug: false
    rpcAddress: 127.0.0.1
    rpcPort: 9890
    rpcPassword: "CC-RPC-PASSWORD"
    rpcDebug: false
    restEnable: false
    restAddress: 127.0.0.1
    restPort: 9990
    restSsl: false
    restSslCertificate: web.crt
    restSslKey: web.key
    restPassword: "PASSWORD"
    restDebug: false
protocols:
    dmr:
        enable: false
        beacons:
            enable: false
            interval: 60
            duration: 3
        control:
            dedicated: false
            enable: false
            slot: 1
            disableGrantSourceIdCheck: false
        disableNetworkGrant: false
        ignoreAffiliationCheck: false
        convNetGrantDemand: false
        legacyGroupReg: false
        embeddedLCOnly: false
        dumpTAData: true
        dumpDataPacket: false
        repeatDataPacket: true
        dumpCsbkData: false
        verifyReg: false
        disableUnitRegTimeout: false
        nRandWait: 8
        backOff: 1
        callHang: 5
        txHang: 8
        silenceThreshold: 21
        frameLossThreshold: 2
        queueSize: 31
        verbose: true
        debug: false
    p25:
        enable: true
        tduPreambleCount: 6
        control:
            enable: true
            ackRequests: true
            dedicated: true
            broadcast: true
            interval: 300
            duration: 1
            notifyActiveTG: false
            disableTSDUMBF: false
            enableTimeDateAnn: false
            disableGrantSourceIdCheck: false
            disableAdjSiteBroadcast: false
            explicitTDUGrantRelease: true
            redundantImmediate: true
            redundantGrantTransmit: false
        controlOnly: false
        disableNetworkHDU: false
        disableNetworkGrant: false
        forceAllowTG0: false
        ignoreAffiliationCheck: false
        immediateCallTerm: true
        inhibitUnauthorized: false
        legacyGroupGrnt: true
        legacyGroupReg: false
        convNetGrantDemand: false
        demandUnitRegForRefusedAff: true
        verifyAff: false
        verifyReg: false
        disableUnitRegTimeout: false
        requireLLAForReg: false
        dumpDataPacket: false
        repeatDataPacket: true
        ignoreDataCRC: false
        dumpTsbkData: false
        callHang: 5
        noStatusAck: false
        noMessageAck: true
        unitToUnitAvailCheck: false
        allowExplicitSourceId: false
        disableDenyResponse: false
        sndcpSupport: false
        silenceThreshold: 124
        frameLossThreshold: 6
        queueSize: 12
        verbose: true
        debug: false
    nxdn:
        enable: false
        control:
            enable: false
            dedicated: false
            broadcast: true
            interval: 300
            duration: 1
            disableGrantSourceIdCheck: false
        ignoreAffiliationCheck: false
        legacyGroupReg: false
        verifyAff: false
        verifyReg: false
        disableUnitRegTimeout: false
        dumpRcchData: false
        callHang: 5
        silenceThreshold: 14
        frameLossThreshold: 4
        queueSize: 31
        verbose: true
        debug: false
system:
    identity: N0CALL-CC
    timeout: 180
    duplex: true
    simplexSameFrequency: false
    modeHang: 10
    fixedMode: false
    rfTalkgroupHang: 10
    activeTickDelay: 5
    idleTickDelay: 5
    localTimeOffset: 0
    disableWatchdogOverflow: false
    info:
        latitude: 0.0
        longitude: 0.0
        height: 1
        power: 10
        location: "Example Site"
    config:
        authoritative: true
        supervisor: false
        channelId: 4
        channelNo: 288
        controlCh:
            rpcAddress: 127.0.0.1
            rpcPort: 0
            rpcPassword: "ULTRA-VERY-SECURE-DEFAULT"
            notifyEnable: true
            presence: 120
        voiceChNo:
            - channelId: 4
              channelNo: 2320
              rpcAddress: 127.0.0.1
              rpcPort: 9891
              rpcPassword: "VC-RPC-PASSWORD"
        secure:
            key: "000102030405060708090A0B0C0D0E0F"
        colorCode: 1
        nac: 293
        ran: 1
        pSuperGroup: FFFE
        announcementGroup: FFFE
        defaultNetIdleTalkgroup: 0
        dmrNetId: 1
        netId: DEE00
        sysId: 123
        rfssId: 1
        siteId: 1
    cwId:
        enable: true
        time: 15
        callsign: N0CALL-CC
    modem:
        protocol:
            type: "uart"
            mode: "air"
            uart:
                port: /dev/ttyUSB0
                speed: 115200
        rxInvert: false
        txInvert: false
        pttInvert: false
        dcBlocker: true
        cosLockout: false
        fdmaPreamble: 80
        dmrRxDelay: 7
        p25CorrCount: 2
        dmrFifoLength: 560
        p25FifoLength: 522
        nxdnFifoLength: 538
        hotspot:
            dmrDiscBWAdj: 0
            dmrPostBWAdj: 0
            p25DiscBWAdj: 0
            p25PostBWAdj: 0
            nxdnDiscBWAdj: 0
            nxdnPostBWAdj: 0
            adfGainMode: 2
            afcEnable: true
            afcKI: 13
            afcKP: 5
            afcRange: 1
            txTuning: 0
            rxTuning: 0
        repeater:
            dmrSymLvl3Adj: 0
            dmrSymLvl1Adj: 0
            p25SymLvl3Adj: 0
            p25SymLvl1Adj: 0
            nxdnSymLvl3Adj: 0
            nxdnSymLvl1Adj: 0
        softpot:
            rxCoarse: 127
            rxFine: 127
            txCoarse: 127
            txFine: 127
            rssiCoarse: 127
            rssiFine: 127
        rxDCOffset: 0
        txDCOffset: 0
        rxLevel: 50
        txLevel: 50
        rssiMappingFile: /opt/dvm/cc/RSSI.dat
        packetPlayoutTime: 10
        disableOFlowReset: false
        ignoreModemConfigArea: false
        dumpModemStatus: false
        displayModemDebugMessages: false
        trace: false
        debug: false
    iden_table:
        file: /opt/dvm/cc/iden_table.dat
        time: 30
    radio_id:
        file: /opt/dvm/cc/rid_acl.dat
        time: 2
        acl: false
    talkgroup_id:
        file: /opt/dvm/cc/talkgroup_rules.yml
        time: 2
        acl: false
EOF
```

### Hotspot 2 — Voice Channel (`/opt/dvm/vc/config.yml`)

Differs from the CC above in: `log` paths, `network.rpcPort`/`rpcPassword`,
**`protocols.p25.control.enable: false`** (this host doesn't run a control channel),
**`system.config.channelNo` (2320)**, **`authoritative: false`**, the **`controlCh`** entry
pointing at the CC (with an empty `voiceChNo` — the VC doesn't grant channels), `system.identity`
/ `cwId.callsign`, and `modem.protocol.uart.port`.

```bash
sudo tee /opt/dvm/vc/config.yml > /dev/null <<'EOF'
daemon: true
iAgreeNotToBeStupid: true
log:
    disableNonAuthoritiveLogging: false
    displayLevel: 1
    fileLevel: 1
    useSyslog: false
    filePath: /opt/dvm/vc
    activityFilePath: /opt/dvm/vc
    fileRoot: DVMVC
network:
    enable: false
    id: 100002
    address: 127.0.0.1
    port: 62031
    password: "PASSWORD"
    encrypted: false
    presharedKey: "000102030405060708090A0B0C0D0E0F000102030405060708090A0B0C0D0E0F"
    jitter: 360
    slot1: true
    slot2: true
    updateLookups: false
    saveLookups: false
    allowActivityTransfer: true
    allowDiagnosticTransfer: true
    allowStatusTransfer: true
    packetDump: false
    debug: false
    rpcAddress: 127.0.0.1
    rpcPort: 9891
    rpcPassword: "VC-RPC-PASSWORD"
    rpcDebug: false
    restEnable: false
    restAddress: 127.0.0.1
    restPort: 9991
    restSsl: false
    restSslCertificate: web.crt
    restSslKey: web.key
    restPassword: "PASSWORD"
    restDebug: false
protocols:
    dmr:
        enable: false
        beacons:
            enable: false
            interval: 60
            duration: 3
        control:
            dedicated: false
            enable: false
            slot: 1
            disableGrantSourceIdCheck: false
        disableNetworkGrant: false
        ignoreAffiliationCheck: false
        convNetGrantDemand: false
        legacyGroupReg: false
        embeddedLCOnly: false
        dumpTAData: true
        dumpDataPacket: false
        repeatDataPacket: true
        dumpCsbkData: false
        verifyReg: false
        disableUnitRegTimeout: false
        nRandWait: 8
        backOff: 1
        callHang: 5
        txHang: 8
        silenceThreshold: 21
        frameLossThreshold: 2
        queueSize: 31
        verbose: true
        debug: false
    p25:
        enable: true
        tduPreambleCount: 6
        control:
            enable: false
            ackRequests: true
            dedicated: false
            broadcast: true
            interval: 300
            duration: 1
            notifyActiveTG: false
            disableTSDUMBF: false
            enableTimeDateAnn: false
            disableGrantSourceIdCheck: false
            disableAdjSiteBroadcast: false
            explicitTDUGrantRelease: true
            redundantImmediate: true
            redundantGrantTransmit: false
        controlOnly: false
        disableNetworkHDU: false
        disableNetworkGrant: false
        forceAllowTG0: false
        ignoreAffiliationCheck: false
        immediateCallTerm: true
        inhibitUnauthorized: false
        legacyGroupGrnt: true
        legacyGroupReg: false
        convNetGrantDemand: false
        demandUnitRegForRefusedAff: true
        verifyAff: false
        verifyReg: false
        disableUnitRegTimeout: false
        requireLLAForReg: false
        dumpDataPacket: false
        repeatDataPacket: true
        ignoreDataCRC: false
        dumpTsbkData: false
        callHang: 5
        noStatusAck: false
        noMessageAck: true
        unitToUnitAvailCheck: false
        allowExplicitSourceId: false
        disableDenyResponse: false
        sndcpSupport: false
        silenceThreshold: 124
        frameLossThreshold: 6
        queueSize: 12
        verbose: true
        debug: false
    nxdn:
        enable: false
        control:
            enable: false
            dedicated: false
            broadcast: true
            interval: 300
            duration: 1
            disableGrantSourceIdCheck: false
        ignoreAffiliationCheck: false
        legacyGroupReg: false
        verifyAff: false
        verifyReg: false
        disableUnitRegTimeout: false
        dumpRcchData: false
        callHang: 5
        silenceThreshold: 14
        frameLossThreshold: 4
        queueSize: 31
        verbose: true
        debug: false
system:
    identity: N0CALL-VC
    timeout: 180
    duplex: true
    simplexSameFrequency: false
    modeHang: 10
    fixedMode: false
    rfTalkgroupHang: 10
    activeTickDelay: 5
    idleTickDelay: 5
    localTimeOffset: 0
    disableWatchdogOverflow: false
    info:
        latitude: 0.0
        longitude: 0.0
        height: 1
        power: 10
        location: "Example Site"
    config:
        authoritative: false
        supervisor: false
        channelId: 4
        channelNo: 2320
        controlCh:
            rpcAddress: 127.0.0.1
            rpcPort: 9890
            rpcPassword: "CC-RPC-PASSWORD"
            notifyEnable: true
            presence: 120
        voiceChNo: []
        secure:
            key: "000102030405060708090A0B0C0D0E0F"
        colorCode: 1
        nac: 293
        ran: 1
        pSuperGroup: FFFE
        announcementGroup: FFFE
        defaultNetIdleTalkgroup: 0
        dmrNetId: 1
        netId: DEE00
        sysId: 123
        rfssId: 1
        siteId: 1
    cwId:
        enable: true
        time: 15
        callsign: N0CALL-VC
    modem:
        protocol:
            type: "uart"
            mode: "air"
            uart:
                port: /dev/ttyUSB1
                speed: 115200
        rxInvert: false
        txInvert: false
        pttInvert: false
        dcBlocker: true
        cosLockout: false
        fdmaPreamble: 80
        dmrRxDelay: 7
        p25CorrCount: 2
        dmrFifoLength: 560
        p25FifoLength: 522
        nxdnFifoLength: 538
        hotspot:
            dmrDiscBWAdj: 0
            dmrPostBWAdj: 0
            p25DiscBWAdj: 0
            p25PostBWAdj: 0
            nxdnDiscBWAdj: 0
            nxdnPostBWAdj: 0
            adfGainMode: 2
            afcEnable: true
            afcKI: 13
            afcKP: 5
            afcRange: 1
            txTuning: 0
            rxTuning: 0
        repeater:
            dmrSymLvl3Adj: 0
            dmrSymLvl1Adj: 0
            p25SymLvl3Adj: 0
            p25SymLvl1Adj: 0
            nxdnSymLvl3Adj: 0
            nxdnSymLvl1Adj: 0
        softpot:
            rxCoarse: 127
            rxFine: 127
            txCoarse: 127
            txFine: 127
            rssiCoarse: 127
            rssiFine: 127
        rxDCOffset: 0
        txDCOffset: 0
        rxLevel: 50
        txLevel: 50
        rssiMappingFile: /opt/dvm/vc/RSSI.dat
        packetPlayoutTime: 10
        disableOFlowReset: false
        ignoreModemConfigArea: false
        dumpModemStatus: false
        displayModemDebugMessages: false
        trace: false
        debug: false
    iden_table:
        file: /opt/dvm/vc/iden_table.dat
        time: 30
    radio_id:
        file: /opt/dvm/vc/rid_acl.dat
        time: 2
        acl: false
    talkgroup_id:
        file: /opt/dvm/vc/talkgroup_rules.yml
        time: 2
        acl: false
EOF
```

Both configs above already set `iAgreeNotToBeStupid: true` and `network.enable: false`
(standalone) — see [step 8](#8-standalone-or-networked) to network them instead. The
`modem.hotspot` tuning values, symbol-level adjustments, and Tx/Rx levels are all left at
their defaults on purpose; that's what [calibration](#6-calibrate-each-independently) is for.

:::{note}
**`iden_table.file`, `radio_id.file`, `talkgroup_id.file`, and `rssiMappingFile` are
absolute paths above on purpose.** `dvmhost` resolves a relative path (the stock
`config.example.yml` just says `iden_table.dat`) against the process's **current working
directory at launch**, not against the directory the config file lives in. Run `dvmhost`
manually from your home directory, or via systemd with no `WorkingDirectory=` set (systemd
defaults to `/`), and a relative `iden_table.dat` either fails to load or silently loads
*some other* file with that name that happens to be sitting in whatever the CWD turned out to
be — which is exactly what produces "the tune menu shows frequencies I never configured."
Absolute paths make this fail safely (or not fail at all) regardless of how or from where you
launch it. If you already deployed a config with the relative form and are seeing wrong
frequencies, this is almost certainly why — redo the `tee` command above, or edit the four
`file:`/`rssiMappingFile` lines in place, then relaunch `--setup` and confirm the tune menu
now shows exactly 451.800000/456.800000 (CC) or 464.500000/469.500000 (VC).
:::

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
WorkingDirectory=/opt/dvm/cc
ExecStart=/opt/dvm/bin/dvmhost -f -c /opt/dvm/cc/config.yml
Restart=on-failure
User=dvm

[Install]
WantedBy=multi-user.target
```

Duplicate as `dvmhost-vc.service` pointing `WorkingDirectory` and `ExecStart` at
`/opt/dvm/vc/config.yml` instead. Start the **CC first**, then the VC:

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
