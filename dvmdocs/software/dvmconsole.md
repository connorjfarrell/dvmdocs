# DVMConsole Software Radio Console

The **Digital Voice Modem Desktop Dispatch Console** ("DDC") is a WPF (Windows) desktop
application that behaves like a traditional dispatch console. It connects to a
[`dvmfne`](dvmhost/dvmfne.md) as a peer and lets an operator monitor and key up on multiple
talkgroups from one screen.

## Features

- Monitor many talkgroups on a DVM FNE simultaneously.
- Dark-mode UI.
- RID → alias mapping so subscriber units show friendly names.
- Encryption-key support for secure talkgroups.
- Codeplug-based configuration.

```{note}
The DDC does **not** interface to base station or mobile radios. For a DVM-compatible console
that does, see [RadioConsole2](https://github.com/W3AXL/RadioConsole2) and
[rc2-dvm](https://github.com/W3AXL/rc2-dvm).
```

## Building

Standard Visual Studio solution; needs .NET and the
[`dvmvocoder`](https://github.com/DVMProject/dvmvocoder) (libvocoder) submodule.

```bash
git clone --recurse-submodules https://github.com/DVMProject/dvmconsole.git
```

Open `dvmconsole.sln`, select the **x86** platform, and build. x64 is supported but
`dvmvocoder` must be compiled separately for x64.

## Configuration

Three files, all referenced from the codeplug:

| File | Purpose |
|------|---------|
| `codeplug.yml` | System parameters, FNE network settings, talkgroup list. Defines the paths to the other two files. An example is in the repo `configs` directory. |
| `keys.clear` | Encryption key entries, each matching a Key ID used in the codeplug. Only needed for encrypted talkgroups. |
| `alias.yml` | Radio ID → display-name mappings. |

Then launch `dvmconsole` and choose **Open Codeplug** to load the configuration.

## Licence

AGPLv3. Amateur / educational use; commercial use strongly discouraged.

## Source

<https://github.com/DVMProject/dvmconsole>
