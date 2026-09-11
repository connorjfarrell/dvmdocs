# Bridging Analog Audio into a DVM Network

[`dvmbridge`](../software/dvmhost/dvmbridge.md) connects analog or PCM audio to a
[`dvmfne`](../software/dvmhost/dvmfne.md), performing realtime vocoding so an analog resource
appears on the digital network as a talkgroup. No modem hardware is involved.

Typical uses: linking an AllStarLink / app_rpt node, a USRP audio stream, or a soundcard
connected to a radio, into a P25/DMR/NXDN talkgroup.

```{mermaid}
graph LR
    ASL["AllStar / app_rpt node"] -->|USRP UDP| USRP["dvmusrp"]
    USRP -->|dvmbridge UDP| BR["dvmbridge"]
    SND["Soundcard / line audio"] -->|PCM| BR
    BR -->|vocoded, peer| FNE["dvmfne"]
    FNE --> H["dvmhost sites"]
```

## Requirements

- The [`dvmvocoder`](https://github.com/DVMProject/dvmvocoder) library.
- An FNE to connect to, and a talkgroup defined for the bridge in `talkgroup_rules.yml`.
- Either a local audio device **or** a UDP audio source.

## Configuration

Edit `bridge-config.yml` (see
[`bridge-config.example.yml`](https://github.com/DVMProject/dvmhost/blob/master/configs/bridge-config.example.yml)):

- **FNE connection** — `peerId`, `address`, `port`, `password` (matching an entry in the FNE's
  `peer_list.dat` if peer ACLs are enabled).
- **Talkgroup / mode** — the digital mode and the source/destination TGID the bridge carries.
- **Audio** — sample format, gain, VOX/COS behavior, and whether audio is local or over UDP.

Set `iAgreeNotToBeStupid: true`.

## Running

### Local soundcard

```bash
dvmbridge -h                       # list audio device IDs
dvmbridge -c bridge-config.yml -i <in-id> -o <out-id>
```

`-i` and `-o` are **required** for local audio. On Windows add `-wasapi` if audio is choppy or
you run many instances.

### USRP / AllStar

AllStar's `chan_usrp` (and similar) speak the **USRP UDP** audio protocol.
[`dvmusrp`](https://github.com/DVMProject/dvmusrp) translates between `dvmbridge` UDP and USRP
UDP:

```text
app_rpt/chan_usrp  <--USRP UDP-->  dvmusrp  <--dvmbridge UDP-->  dvmbridge  <-->  dvmfne
```

Configure the UDP ports to line up across the three processes; run `dvmusrp` and `dvmbridge`
on the same host (or a trusted link) since the audio between them is unauthenticated.

## Tips

- One `dvmbridge` instance carries one bridge. Run multiple instances for multiple talkgroups.
- Watch levels — over-deviated analog audio into the vocoder produces poor digital audio just
  as it does on RF.
- For a full analog/digital console with radio control instead of a plain bridge, see
  [RadioConsole2 / rc2-dvm](https://github.com/W3AXL/rc2-dvm).

## Source / further reading

- `dvmbridge` section of the `dvmhost` README: <https://github.com/DVMProject/dvmhost/blob/master/README.md>
- `dvmusrp`: <https://github.com/DVMProject/dvmusrp>
- `dvmvocoder`: <https://github.com/DVMProject/dvmvocoder>
