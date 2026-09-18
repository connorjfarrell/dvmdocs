# Channel Identity Tables (`iden_table.dat`)

The **channel identity table** is the first thing to get right when setting up any DVM system,
and the most common source of "it transmits on the wrong frequency" problems. This guide
explains what it is, how `dvmhost` uses it, and how to generate entries.

## What it does

`dvmhost` never stores a raw frequency for a channel. Instead it stores a **channel identity**
(`channelId`) plus a **channel number** (`channelNo`), and computes the frequency from the
matching row of `iden_table.dat`.

Those computed frequencies are used for:

- the operating frequency of a hotspot,
- the Rx/Tx frequency split of a repeater or modem,
- the over-the-air messages that tell trunking radios which {term}`VC` to move to.

If the table is wrong, every one of those is wrong.

## File format

`iden_table.dat` is a plain CSV. Each row:

```text
# ChId, Base Freq (Hz), Spacing (kHz), Input (Tx) Offset (MHz), Bandwidth (kHz),
0,851006250,6.25,-45.000,12.5,
1,762006250,6.25,30.000,12.5,
2,450000000,6.25,5.000,12.5,
3,146000000,6.25,1.000,12.5,
15,935001250,6.25,-39.00000,12.5,
```

| Field | Meaning |
|-------|---------|
| **ChId** | Channel identity number, referenced by `channelId` in `config.yml`. P25 allows identities 0–15. |
| **Base Freq** | The frequency (Hz) of channel number 0 for this identity. |
| **Spacing** | Channel spacing in kHz. The receive frequency of channel *n* is `Base Freq + (n × Spacing)`. |
| **Input Offset** | Transmit (radio input) offset in MHz, applied relative to the receive frequency. Negative for standard duplex splits. |
| **Bandwidth** | Channel bandwidth in kHz (12.5 for P25 Phase 1 / DMR / NXDN). |

So for identity `3` above (`Base 146000000`, spacing `6.25 kHz`, offset `+1.000 MHz`), thinking
purely in raster steps:

- step `0` → radio Rx 146.00000 MHz, radio Tx 147.00000 MHz
- step `160` → 146.00000 + (160 × 6.25 kHz) = 147.00000 MHz Rx, 148.00000 MHz Tx

```{note}
"Radio Rx / radio Tx" here is from the **subscriber's** point of view. The site (repeater)
transmits on the radio's Rx frequency and vice-versa.
```

```{warning}
**`channelNo` (and `rxChannelNo`) in `config.yml` are parsed as hexadecimal text, not
decimal** — confirmed directly from `dvmhost`'s source
(`src/host/Host.Config.cpp`): `::strtoul(rfssConfig["channelNo"].as<std::string>("1").c_str(),
NULL, 16)`. This is the *same* rule `nac`, `sysId`, `netId`, `rfssId`, `siteId`, and `dmrNetId`
already follow — P25 identifiers are conventionally hex — but it's easy to miss for
`channelNo` specifically, because a raster step count doesn't *look* like a P25 identifier.

**The raster-step math above stays in decimal.** Once you have a step count, convert *that*
number to its hexadecimal digits and write those digits as the YAML value. Step `160` decimal
is `0xA0` — so you write `channelNo: A0` (or `channelNo: 160` if the value you land on happens
to *already* be what you want when read back as hex, which is easy to get wrong by accident —
always do the conversion explicitly rather than trust a coincidence). Get this backwards and
`dvmhost` will run with a real, different, silently-wrong frequency — not an error, not a
crash, just the wrong channel — because `strtoul` happily parses ordinary-looking decimal
digits as hex without complaint.
```

## Configuring a channel

In `config.yml`, the channel-step value from the raster math above is written as **hex text**:

```yaml
system:
  config:
    channelId: 3      # row in iden_table.dat — plain decimal, NOT hex
    channelNo: A0      # step 160 (decimal) written as hex "A0" — NOT "160"
```

(`channelId` is the one exception to the hex rule — it's read with a plain decimal parse, so
`channelId: 3` really does mean channel identity 3.)

For trunking sites, the control-channel host and each voice-channel host use their own
`channelId` / `channelNo`, and the CC host also lists every VC under
`system.config.voiceChNo` (whose `channelNo` follows the same hex rule). All hosts at a site
must share the **same identity table**.

### Explicit (non-standard-split) channels

If a voice channel's Tx/Rx split doesn't follow the identity's `Input Offset`, define it
explicitly with `rxChannelId` / `rxChannelNo` alongside `channelId` / `channelNo` in the
`voiceChNo` entry — `channelId`/`channelNo` then represent the transmit side.

## Generating entries

Use the [`iden-channel-calculator`](https://github.com/DVMProject/iden-channel-calculator)
(a rewrite of the old `iden_channel_calc.py` shipped in `dvmhost`):

```bash
pip install git+https://github.com/DVMProject/iden-channel-calculator.git

# channel number -> frequency
iden-channel-calculator -b 144000000 -c 100

# frequency -> channel number
iden-channel-calculator -b 144000000 -t 145600000

iden-channel-calculator -h   # channel spacing and other options
```

Pick a **Base Freq** at or below the lowest frequency you'll use, choose a spacing that
divides your channel plan evenly (6.25 kHz is a safe universal choice), then compute the
channel step for each operating frequency. The calculator's output is a plain decimal step
count — convert it to hex before writing it into `config.yml`'s `channelNo` (see the warning
above).

## Source / further reading

- Example file: <https://github.com/DVMProject/dvmhost/blob/master/configs/iden_table.example.dat>
- Calculator: <https://github.com/DVMProject/iden-channel-calculator>
- Host config reference: [`config.example.yml`](https://github.com/DVMProject/dvmhost/blob/master/configs/config.example.yml)
