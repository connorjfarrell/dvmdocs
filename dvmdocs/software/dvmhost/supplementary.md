# Supplementary `dvmhost` Applications

Beyond the core applications, the `dvmhost` repository builds several utilities for patching
traffic and for monitoring / editing configuration.

## `dvmpatch` — talkgroup patching

`dvmpatch` manually patches together talkgroups **of the same digital mode** so that traffic
on one is repeated onto the other(s). It connects to an [FNE](dvmfne.md) as a peer.

```
usage: ./dvmpatch [-vhf] [-c <configuration file>]
```

Configured with `patch-config.yml`; see
[`patch-config.example.yml`](https://github.com/DVMProject/dvmhost/blob/master/configs/patch-config.example.yml).
The config defines the FNE connection and the source/destination talkgroups to patch.

## Monitoring and editor TUIs

These are only built when the project is compiled **with** TUI support
(`-DENABLE_TUI_SUPPORT` not set to `0`).

| Tool | Purpose |
|------|---------|
| `dvmmon` | Semi-realtime console monitor for one or more `dvmhost` instances. See also the separate [`dvmhost_monitor`](https://github.com/DVMProject/dvmhost_monitor) repository. |
| `sysview` | Near-realtime console monitor for a `dvmfne` instance (peers, calls, affiliations). |
| `tged` | Editor for `talkgroup_rules.yml` talkgroup-rules files. |
| `peered` | Editor for `peer_list.dat` peer-list data files. |

Each is launched directly from the install `bin` directory and connects to the relevant
instance's RPC/REST endpoint or operates on the named file.

## Scripting

For programmatic access to running instances, use
[`pydvm`](https://github.com/DVMProject/pydvm) (Python) or [`dvmcmd`](dvmcmd.md) (shell)
against the REST API.
