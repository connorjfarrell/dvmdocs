# Quickstart: Dual-Hotspot P25 Trunked Node (x86)

```{warning}
**Work in progress.** Firmware selection (step 3) and the identity-table math (step 4) have
been checked against real hardware bring-up; networking, calibration, and full channel-grant
operation have not yet been confirmed end-to-end. Cross-check against your board's own
documentation, and see something wrong? Fix it — see [Contributing](../contributing.md).
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
- Two USB-connected `MMDVM_HS_Dual` boards, each flashed with
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
git clone --recurse-submodules https://github.com/DVMProject/dvmfirmware-hs.git
cd dvmfirmware-hs
make -f Makefile.STM32FX mmdvm-hs-hat-dual
```

This builds the **`master`** branch, producing a plain `dvm-firmware-hs_f1.bin` flashed as a
single image at the default address — correct for `MMDVM_HS_Dual` boards using an external
USB-serial bridge chip (CP210x/CH340/FTDI — i.e. anything showing up as `/dev/ttyUSB*`, which
is what this guide assumes throughout).

```{note}
If your board instead enumerates as `/dev/ttyACM*` — meaning the STM32 itself implements USB,
with no separate bridge chip (`lonestar-usb` is the named example) — you need the
`usb-support` branch's bootloader-enabled build and a two-stage flash instead. That's a
different procedure; see the `dvmfirmware-hs` repository for that path.
```

Both boards are the same `MMDVM_HS_Dual` type, so build once and flash it to both. Confirm
which board is on which labeled port (per the [Device names](#device-names) note above) before
you start:

1. Put the board in bootloader mode — jumper **BOOT0** (sometimes labeled **J1**) per your
   board's documentation; boards vary here, so check the seller's instructions.
2. Flash over its device:
   ```bash
   sudo stm32flash -v -w dvm-firmware-hs_f1.bin -R /dev/ttyUSB0   # CC board
   sudo stm32flash -v -w dvm-firmware-hs_f1.bin -R /dev/ttyUSB1   # VC board
   ```
3. Remove the boot jumper and power-cycle (or `-R` above already resets it). The board's
   heartbeat LED (commonly labeled **D0**) should blink steadily once it's running normal
   firmware — confirm that before moving on.

:::{note}
**`dvmhost` reports "could not communicate with modem," even though the device path, firmware,
and heartbeat LED all check out?** Check, in order:

1. **BOOT0 jumper still fitted.** The board stays in the ST bootloader (which speaks the
   `stm32flash` protocol, not the DVM modem protocol) and won't respond to `dvmhost`. Remove
   it and power-cycle.
2. **The heartbeat LED stops the moment `dvmhost` (or anything) opens the port.** This is a
   strong sign the port open is **resetting the MCU** — many of these boards wire DTR or RTS
   into a reset/bootloader-select circuit (the same mechanism used to enter the bootloader in
   software, e.g. for `dvmhost --boot`), and if the serial driver asserts that line on open,
   it can hold the chip in reset or drop it back into the bootloader instead of leaving the
   application running. To confirm: open the port with a plain terminal
   (`screen /dev/ttyUSB0 115200` or `minicom`) and watch whether the LED stops the same way —
   if it does, it's not `dvmhost`-specific. Things to try: check the board for a jumper that
   disables auto-reset-on-DTR (naming varies by vendor); explicitly clear DTR/RTS on the port
   before starting `dvmhost`, e.g.
   ```bash
   python3 -c "import serial; s = serial.Serial('/dev/ttyUSB0'); s.dtr = False; s.rts = False"
   ```
   (`python3-serial` package); or `stty -F /dev/ttyUSB0 115200 -hupcl clocal` to stop the
   port from dropping DTR on close, which can help if it's the close/reopen cycle triggering
   the reset rather than the open itself.
3. **ModemManager or `brltty` has the port open.** Both are common on desktop Ubuntu/Debian
   installs and grab newly-plugged USB-serial devices to probe them. The
   `ENV{ID_MM_DEVICE_IGNORE}="1"` udev rule from step 1 handles ModemManager; if `brltty` is
   installed and you don't need it, `sudo systemctl disable --now brltty.service`.
   `sudo lsof /dev/ttyUSB0` shows you if something else already has the port open.
4. **Wrong baud rate.** Must be `115200` unless you changed it in the firmware.
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

- **CC** (Hotspot 1) at 451.800 MHz: step `= (451.800 − 450.000) MHz / 6.25 kHz = 1800 kHz / 6.25 kHz = 288`
- **VC** (Hotspot 2) at 464.500 MHz: step `= (464.500 − 450.000) MHz / 6.25 kHz = 14500 kHz / 6.25 kHz = 2320`

```{warning}
**`config.yml`'s `channelNo` field is parsed as hexadecimal text, not decimal** — confirmed
directly from `dvmhost`'s source (`src/host/Host.Config.cpp`):
`::strtoul(rfssConfig["channelNo"].as<std::string>(...).c_str(), NULL, 16)`. Same rule as
`nac` / `sysId` / `netId` / `rfssId` / `siteId` (all conventionally hex for P25), just easy to
miss here since a raster step count doesn't look like a P25 identifier. The step counts above
are decimal — convert **each one** to hex before it goes in `config.yml`:

- CC step `288` decimal → **`120`** hex → `channelNo: 120`
- VC step `2320` decimal → **`910`** hex → `channelNo: 910`

Get this backwards (write the decimal digits directly) and `dvmhost` runs on a real, different,
silently wrong frequency — no error, no crash, just the wrong channel. See the
[Channel Identity Tables guide](iden-tables.md) for the general rule.
```

→ `channelId: 4`, `channelNo: 120` (CC) → output 451.800000 MHz, input 456.800000 MHz
→ `channelId: 4`, `channelNo: 910` (VC) → output 464.500000 MHz, input 469.500000 MHz

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
    identity: N0CALLCC
    timeout: 180
    duplex: true
    simplexSameFrequency: false
    modeHang: 10
    fixedMode: true
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
        channelNo: 120
        controlCh:
            rpcAddress: 127.0.0.1
            rpcPort: 0
            rpcPassword: "ULTRA-VERY-SECURE-DEFAULT"
            notifyEnable: false
            presence: 120
        voiceChNo:
            - channelId: 4
              channelNo: 910
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
        callsign: N0CALLCC
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
    identity: N0CALLVC
    timeout: 180
    duplex: true
    simplexSameFrequency: false
    modeHang: 10
    fixedMode: true
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
        channelNo: 910
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
        callsign: N0CALLVC
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
launch it. If the setup TUI's tune menu ever shows frequencies you didn't configure, check
these four paths first.
:::

### Radio ID ACL and talkgroup rules

Both configs above point at `rid_acl.dat` and `talkgroup_rules.yml` on each host, but neither
file exists yet — `dvmhost` will error (`Cannot open the radio ID lookup file` /
`Cannot open the talkgroup rules lookup file`) without them. `radio_id.acl` and
`talkgroup_id.acl` are both `false`, so neither file needs to *restrict* anything — but
`talkgroup_rules.yml` isn't just an access-control file: it's what defines a talkgroup as
existing on the system at all. Without at least one entry, there's nothing for a radio to
affiliate to or a call to grant onto.

```bash
sudo touch /opt/dvm/cc/rid_acl.dat
sudo touch /opt/dvm/vc/rid_acl.dat

sudo tee /opt/dvm/cc/talkgroup_rules.yml > /dev/null <<'EOF'
groupVoice:
  - name: Talkgroup 1
    alias: Test
    config:
      active: true
      affiliated: false
      inclusion: []
      exclusion: []
      rewrite: []
      always: []
      preferred: []
      rid_permitted: []
    source:
      tgid: 1
      slot: 1
EOF

sudo cp /opt/dvm/cc/talkgroup_rules.yml /opt/dvm/vc/talkgroup_rules.yml
```

## 6. Calibrate each independently

`MMDVM_HS_Dual` boards have **no trimpots** — every Tx/Rx adjustment is a software value in
`config.yml`, set interactively through `dvmhost`'s built-in setup TUI and then saved. This
section is a full worked alignment for both boards against a **P25-capable service monitor** —
one with real FM deviation measurement, a frequency counter, and native P25 Phase 1 decode and
generation (an R8200-class unit, or equivalent). If your monitor is only a bare spectrum
analyzer with no deviation meter or P25 decode, see the note at the end of this section for a
fallback method — but if you have real P25 capability, use it; it's both simpler and more
accurate than working around its absence.

```{warning}
This exact sequence hasn't been run end-to-end against real hardware yet — the config
structure, frequencies, and channel math earlier in this guide have been (see the warning at
the top of this page), but the calibration walkthrough below is the next thing to validate.
Treat specific numbers (target deviation, starting levels, sensitivity target) as a starting
point, not gospel.
```

### What you're actually adjusting

There is no physical alignment on these boards — everything lives in `system.modem` in
`config.yml`, written by the setup TUI:

| Config field | What it controls | Starting value |
|---|---|---|
| `txLevel` | Transmit deviation level (software "pot"). | `50` |
| `rxLevel` | Receive gain/level (software "pot"). | `50` |
| `system.modem.hotspot.txTuning` | Transmit frequency trim, in **Hz**. | `0` |
| `system.modem.hotspot.rxTuning` | Receive frequency trim, in **Hz**. | `0` |
| `system.modem.hotspot.adfGainMode` | ADF7021 Rx LNA gain (`0` Auto, `1` Auto High Lin, `2` Low, `3` High). | `2` |
| `system.modem.hotspot.afcEnable` | ADF7021 automatic frequency correction. | `true` |

You'll set these per board (CC and VC each get their own values — they're physically
different chips) by running the setup TUI, adjusting until the numbers below look right, and
then **File → Save Settings** before quitting — that's what actually writes them back into
`config.yml`.

### Prerequisites

- Both boards flashed, wired, and powered; `config.yml` for the board you're calibrating
  loaded and correct (step 5).
- A cable from the board's antenna connector to the service monitor's RF In/Out (or a suitable
  attenuator — don't feed a monitor's front end directly from a Tx output above its rated
  input).
- The service monitor's own P25-capable RF generator, for the Rx half.

### Calibrating the CC (451.800 / 456.800 MHz)

Start the setup TUI:

```bash
/opt/dvm/bin/dvmhost -c /opt/dvm/cc/config.yml --setup
```

**Step 1 — zero everything.** **Calibrate → Level Adjustment** (or **F5**): set `TX Level` and
`RX Level` to `50`. In the same session, set `Tx Tuning` and `Rx Tuning` to `0`. This is your
known-good starting point every time.

**Step 2 — frequency error, first.**
1. **Calibrate → Operational Mode**, select **[Tx] P25 1011 Hz Test Pattern (NAC293 ID1
   TG1)** — substitute your own NAC if you changed it from the example `293`. This is the
   actual ANSI/TIA-102.CAAA reference test pattern, not an arbitrary signal — it's what the
   whole rest of this procedure is built around.
2. On the service monitor: frequency-error/counter mode, centered on **451.800000 MHz**.
3. Key up (**F12**), read the error, un-key.
4. Enter the **opposite** value into `Tx Tuning` (e.g. `+180 Hz` error → `Tx Tuning: -180`).

Doing frequency first means the deviation reading in the next step isn't being taken on a
slightly off-center carrier.

**Step 3 — deviation, off the P25 decode.**
1. Same pattern, still keyed on 451.800000 MHz.
2. Use the monitor's **C4FM/P25 modulation analysis mode** — not a generic wideband FM
   deviation meter — to read the peak (outer, ±3) symbol deviation. If it reports a
   per-symbol or modulation-fidelity breakdown, use that; it reflects what an actual P25
   receiver's symbol slicer sees, which a single generic deviation number doesn't.
3. Adjust `TX Level` until peak deviation lands **2.75–2.83 kHz** (2.80 kHz target; a little
   high, ~2.9 kHz, is acceptable and trades off slightly worse BER).

**Step 4 — verify by decode, not just by the numbers.** With the monitor still receiving the
same pattern, confirm it actually **decodes** cleanly — correct NAC, no or low frame errors. A
deviation reading that "looks right" but doesn't decode cleanly means something else is off
(symbol-level imbalance, filtering) that a deviation number alone won't catch. Un-key.

**Step 5 — repeat once.** Steps 2–4 once more — frequency and deviation interact slightly at
the margins, so a second pass tightens both up.

**Step 6 — Rx frequency trim.** As a starting point, set `Rx Tuning` to the **same value** you
landed on for `Tx Tuning` — Rx trim usually tracks Tx trim closely on these boards. You'll
fine-tune it against BER next.

**Step 7 — Rx level via BER.**
1. On the service monitor: RF generator mode, set to **456.800000 MHz** (the CC's *input* —
   what a radio transmits back to the site on), generating the actual TIA-102.CAAA 1011 Hz
   pattern (not analog FM) at the correct NAC and 2.75–2.83 kHz deviation — the value you just
   calibrated on Tx is a good reference, since a decent generator is a "gold standard" source.
   Start at a moderate output level, e.g. −60 dBm.
2. On the `dvmhost` setup TUI: **Calibrate → Operational Mode**, select **[Rx] P25 1011 Hz
   Test Pattern**. The **Receive BER** window (top-right of the TUI) shows live BER while a
   signal is present.
3. Adjust `RX Level` (and, if BER won't settle, `Rx Tuning` in small steps around the value
   from Step 6) for the **lowest BER** at this level.

**Step 8 — measure real sensitivity, don't just eyeball one BER number.** This is the step
most guides skip. Sweep the generator's output level down in 3–6 dB steps and log BER at each
one. Find where BER crosses **5 %** — the conventional P25 digital sensitivity benchmark (the
digital analog of 12 dB SINAD for analog FM). That crossing point is your site's actual
measured Rx sensitivity (e.g. "−114 dBm @ 5 % BER") — a real number you can record and compare
against typical ADF7021 hotspot performance, rather than an untested assumption that "it works
at −60 dBm." If BER stays flat-bad across the whole sweep instead of degrading gracefully as
level drops, that's a level/tuning problem, not a sensitivity problem — recheck Steps 6–7. Try
toggling `adfGainMode` between `2` (Low) and `0` (Auto) if you suspect front-end overload or
desense from the nearby Tx chip.

**Step 9 — save.** **File → Save Settings**, then **File → Quit**. Confirm the values landed
in `/opt/dvm/cc/config.yml`.

### Calibrating the VC (464.500 / 469.500 MHz)

Identical procedure, different config and frequencies:

```bash
/opt/dvm/bin/dvmhost -c /opt/dvm/vc/config.yml --setup
```

- Tx frequency/deviation/decode steps: same test pattern, service monitor tuned to
  **464.500000 MHz** instead of 451.800000 MHz.
- Rx BER + sensitivity sweep: service monitor generating on **469.500000 MHz** instead of
  456.800000 MHz.
- Everything else (target 2.80 kHz peak deviation, starting `TX/RX Level` of 50, Rx Tuning
  following Tx Tuning, the 5 % BER sensitivity benchmark) is identical.

```{note}
Do this **one board at a time** — running both Tx simultaneously while reading BER on the
other will just confuse you, and calibrate the **CC first**: you need it transmitting control
data before a radio will show you anything meaningful on the VC at all.
```

```{note}
**No deviation meter or P25 decode, only a bare spectrum analyzer?** Fall back to a Bessel-null
technique: transmit `[Tx] P25 1200 Hz Tone Mode`, and instead of measuring deviation directly,
adjust `TX Level` until the **center carrier nulls** on the spectrum display (symmetric side
lobes, clean tone) — for a 1200 Hz tone this occurs near the first Bessel zero, close to the
2.75–2.83 kHz target, giving you an absolute deviation reference without a calibrated meter.
Then proceed to frequency error and Rx exactly as above. This is a coarser substitute for
Steps 2–4 above, not an additional step to do in combination with them.
```

### Duplex self-desense

Because Tx and Rx are separate ADF7021 chips sharing one small board (and often one antenna
port), it's worth confirming the Tx chip isn't desensing the Rx chip: with the CC transmitting
continuously (its normal state once running), re-check Rx BER (or re-run the sensitivity
sweep) on the VC and on the CC itself. A meaningful degradation while the CC's Tx is active
suggests you need more physical separation or filtering between the boards/antennas, not a
software fix — `adfGainMode`/`afcEnable` tuning can only compensate for so much front-end
overload.

### Cross-check with a real radio

Once both boards are calibrated and running (step 9), the real test is a P25 radio programmed
for this system (WACN `DEE00`, System ID `123`, CC output/input 451.800/456.800 MHz, talkgroup
1 from the [talkgroup rules created above](#radio-id-acl-and-talkgroup-rules)): confirm it
registers on the CC, then key up on TG 1 and confirm it's granted to and audible on
464.500/469.500 MHz. If the radio has its own BER/Tuner test mode, use it the same way you
used the service monitor above for a final sanity check from the subscriber's perspective.

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
