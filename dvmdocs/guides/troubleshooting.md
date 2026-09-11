# Troubleshooting & FAQ

Common problems and where to look. When asking for help on the
[Discord](https://discord.gg/3pBe8xgrEz), include your `config.yml` (redact passwords), the
board you're using, and the relevant log lines with `log.displayLevel: 1`.

## Modem / serial

**"Cannot open the serial port" / modem won't connect**
: Wrong `system.modem.protocol.uart.port`, or the port is in use. On a Raspberry Pi you must
  free `/dev/ttyAMA0` first — disable Bluetooth and the serial console
  ([Pi preparation](../software/dvmhost/installation.md#raspberry-pi-preparation)). Check the
  user running `dvmhost` is in the `dialout` group.

**Modem connects but reports the wrong protocol/firmware, or nothing decodes**
: You are running stock MMDVM firmware. DVM requires the
  [DVMProject firmware forks](../software/dvmfirmware/index.md).

**DVM-V24 connects then immediately drops**
: Firmware must be **≥ 2.0**. Check the `CLKSEL` jumper is fitted. If `UBT0`/`URST` are
  bridged the board resets on every connect — remove them for normal use.

**Modem keeps resetting ("ADC/DAC overflow")**
: Rx or Tx level far too high. Restart calibration with levels at 50 and hardware pots at
  minimum. As a last resort `system.modem.disableOFlowReset: true` masks the symptom.

## RF / audio quality

**BER consistently above 10 %**
: Almost always frequency error. Keep the radio's reference within ±150 Hz. Also check for DC
  offset in the audio path and that FM deviation is 2.75–2.83 kHz (not the 2.5 kHz a
  commercial 12.5 kHz alignment enforces — you may need to de-tune the radio).

**Works on P25 but not DMR (or vice-versa)**
: Symbol levels may need per-mode trimming (`system.modem.repeater.*SymLvl*Adj`) — only with
  RF test equipment. Check the mode is actually enabled under `protocols:`.

**Hotspot: Rx desense in trunking mode**
: Set `system.modem.hotspot.adfGainMode: 2` (Low) and consider a 90° antenna adapter to
  decouple Tx and Rx.

## Networking / FNE

**Startup warning: "Could not resize socket recv/send buffer"**
: Raise `net.core.rmem_max` and `net.core.wmem_max` to at least `524288` (see your distro's
  sysctl docs).

**Host won't connect to the FNE**
: Check `network.id` is unique, the `password` matches, and — if peer ACLs are on — the peer
  ID is in the FNE's `peer_list.dat`. Open **both** `62031` and `62032` UDP. If
  `network.encrypted: true`, the `presharedKey` must match exactly on both ends.

**Talkgroup is silent on some sites**
: `talkgroup_rules.yml`: check `active: true`, and whether `affiliated: true` is suppressing
  it to sites with no affiliated radio. Check `inclusion` / `exclusion` peer lists.

**Local ACL / talkgroup rules keep reverting**
: The FNE is pushing rules and your host is overwriting them with its local file. Set the
  host's `radio_id.time` / `talkgroup_id.time` to `0`.

## Trunking

**Radios see the control channel but calls never grant**
: Check the `iden_table.dat` is identical across CC and VC hosts and produces the right
  frequencies. Verify the CC's `system.config.voiceChNo` RPC settings match each VC's
  listener, and each VC's `controlCh` points back at the CC.

**Radios keep roaming away / get AFF_GRP_RSP DENY**
: The talkgroup's `preferred` list in `talkgroup_rules.yml` doesn't include this site's CC
  peer ID. Empty list = all sites preferred.

## Build

**`jupyter-book` errors on this docs repo** (contributors)
: This book is Jupyter Book **v1**. Install `jupyter-book<2` — see [Contributing](../contributing.md).

**`dvmhost` build: ASIO not found when cross-compiling**
: Download ASIO manually and pass `-DWITH_ASIO=/path/to/asio` (required for ARM 32-bit).

## Still stuck?

- Turn on `verbose: true` / `debug: true` for the relevant protocol and re-read the log.
- Check the technical notes in <https://github.com/DVMProject/dvmhost/tree/master/docs>.
- Ask on Discord: <https://discord.gg/3pBe8xgrEz> (remember the
  [usage policy](../overview/introduction.md) — public-safety use gets no support).
