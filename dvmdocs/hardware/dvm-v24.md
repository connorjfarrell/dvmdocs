# DVM-V24 Motorola V.24 USB Adapter

The **DVM-V24** adapts the legacy Motorola **V.24 synchronous serial** interface — used by
base stations and comparators such as the Quantar and AstroTAC — directly to USB and the
[`dvmhost`](../software/dvmhost/dvmhost.md) application. It replaces the traditional stack of
Cisco routers and WIC cards previously needed to link a Quantar to a network.

```{note}
The DVM-V24 is **P25 only** and uses `dvmhost`'s `dfsi` modem mode. Firmware **revision 2.0 or
greater is required** for use with `dvmhost`.
```

## Where to buy

Boards are available from the
[W3AXL Online Store](https://store.w3axl.com/products/dvm-v24-v2-usb-converter-for-v-24-equipment).
Schematics are in the repository `hw/` directory for building your own.

## Hardware revisions

| Rev | Notes |
|-----|-------|
| **V1** | Original design. Requires the `CLKSEL` jumper to be fitted. Some units experienced USB lockup/freezing. Must be flashed via ST-Link/SWD. |
| **V2** | USB↔serial handled by a dedicated **CP2102** (fixes the lockups). `CLKSEL` solder jumper is shorted by default. Adds `UBT0` / `URST` jumpers and supports USB serial-bootloader flashing. |

The two revisions need different firmware binaries (both built from the same repo).

### Jumpers

- **`CLKSEL`** — connects the serial clock to the `RXCLK` pin. Must be in place; the firmware
  currently generates clocks for both Tx and Rx. (Fitted by solder jumper on V2.)
- **`UBT0` / `URST`** (V2 only) — route the serial chip's RTS/DTR to force UART bootloader
  mode, allowing `stm32flash` programming without the `dvmhost` boot command. **Not** bridged
  by default. When bridged, the board resets whenever `dvmhost` connects.

## Firmware

Bare-C firmware generated from STM32CubeMX, in the repo `fw/` directory. Build on Linux:

```bash
sudo apt install gcc-arm-none-eabi cmake
mkdir build && cd build
cmake ..
make            # builds both v1 and v2 (or: make dvm-v24-v1 / make dvm-v24-v2)
```

### Flashing

**ST-Link / SWD** (required for V1, and for V2 recovery):

```bash
sudo apt install stlink-tools
st-flash --reset write dvm-v24-xxx.bin 0x8000000
```

**USB serial bootloader** (V2): put the board in bootloader mode via
`dvmhost -c <config>.yml --cal` then the `!` command, then:

```bash
stm32flash -v -w ./dvm-v24-v2.bin -R /dev/ttyUSBx
# or, with UBT0/URST bridged:
stm32flash -v -w ./dvm-v24-v2.bin -i 'rts&-dtr:-rts&dtr' /dev/ttyUSBx
```

## Quantar connection

Use a **straight-through** RJ45 (ordinary Ethernet) cable from the DVM-V24 to the Quantar's
front wireline connectors. Pinout:
[diagram in repo](https://github.com/DVMProject/dvmv24/blob/main/pics/pinout.png).

### Quantar RSS settings

| Field | Value |
|-------|-------|
| Wireline Operation | 4 WIRE FULL DUPLEX |
| Astro To Wireline | ENABLED |
| Wireline Interface | V.24 ONLY |
| External Transmit Clock | ENABLED |
| RT/RT Configuration | DISABLED |

### `dvmhost` configuration

```yaml
modem:
    protocol:
        type: "uart"
        mode: "dfsi"
        uart:
            port: /dev/ttyACM0   # your V24 board's serial port
            speed: 115200
    dfsi:
        rtrt: true
        diu: true
        jitter: 200
```

Additional `dfsi:` options (`callTimeout`, `fullDuplex`, and the UDP/`fsc` mode for networked
V.24) are documented in
[`config.example.yml`](https://github.com/DVMProject/dvmhost/blob/master/configs/config.example.yml)
and `docs/TN.1001 - Modem Protocol.adoc`.

```{note}
CCGW V.24 compatibility is still being investigated; in Quantar-compatibility mode the CCGW
does not fully mirror the Quantar's V.24 behavior.
```

## Source / support

- Repository: <https://github.com/DVMProject/dvmv24>
- Discord: <https://discord.gg/3pBe8xgrEz>
