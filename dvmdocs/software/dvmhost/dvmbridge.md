# `dvmbridge` digital-to-analog gateway bridge

`dvmbridge` connects analog or PCM audio resources to a [`dvmfne`](dvmfne.md) instance,
performing realtime vocoding in both directions. It is how you link analog systems — an AllStar
node, a USRP audio stream, or a soundcard/line interface — into a DVM digital network without
any modem hardware.

It connects to the FNE as a peer, so from the network's point of view a bridge behaves like
another site carrying one or more talkgroups.

## Requirements

- The [`dvmvocoder`](https://github.com/DVMProject/dvmvocoder) library (IMBE/AMBE vocoding).
- An FNE to connect to.
- For local audio: an input and output audio device.

## Configuration

Configured with `bridge-config.yml`; see the annotated
[`bridge-config.example.yml`](https://github.com/DVMProject/dvmhost/blob/master/configs/bridge-config.example.yml).
It defines the FNE connection (peer ID, address, password), the talkgroup(s) to bridge, the
digital mode, and audio/vocoder parameters. There is no configuration beyond this file plus
the command-line device selection.

## Running `dvmbridge`

```
usage: ./dvmbridge [-vhf] [-i <input audio device id>] [-o <output audio device id>]
                   [-wasapi] [-c <configuration file>]

  -v        show version information
  -h        show help
  -f        foreground mode
  -i        input audio device id
  -o        output audio device id
  -wasapi   use WASAPI instead of WinMM (Windows only)
  -c <file> configuration file to use
```

- If using local audio, `-i` and `-o` are **required**. Run `dvmbridge -h` to list the
  available device IDs.
- On Windows `dvmbridge` defaults to WinMM. Use `-wasapi` for the high-performance audio path,
  which can help when running many bridge instances or when audio is choppy.

## USRP interop

To bridge to systems that speak the USRP UDP audio protocol (e.g. some AllStar/ASL setups),
use [`dvmusrp`](https://github.com/DVMProject/dvmusrp), a helper that converts `dvmbridge` UDP
audio to/from USRP UDP.

## Source

- Annotated config: [`bridge-config.example.yml`](https://github.com/DVMProject/dvmhost/blob/master/configs/bridge-config.example.yml)
- README: <https://github.com/DVMProject/dvmhost/blob/master/README.md>
