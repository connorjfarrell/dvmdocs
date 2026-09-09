# `dvmfne` Fixed-Network Equipment Routing Core

`dvmfne` is a network server that interconnects DVM endpoints. `dvmhost` instances, consoles,
and `dvmbridge` instances connect to an FNE as **downstream peers**; other FNEs connect as
**peer links**. The FNE switches voice and data traffic between them according to its
talkgroup rules.

```{warning}
Like the rest of the suite, `dvmfne` is for amateur and educational use only and must never be
used in public safety or life safety critical applications.
```

## Concepts

- **Peer ID** — every peer (host, bridge, console, linked FNE) has a globally unique numeric
  ID. The FNE's own `master.peerId` must also be unique across all connected networks.
- **Talkgroup rules** — `talkgroup_rules.yml` defines every valid talkgroup and its routing /
  access-control parameters. This is the single most important FNE file after the main config.
- **Peer links & spanning tree** — multiple FNEs can be meshed. An enabled spanning tree
  (`master.enableSpanningTree`) prevents traffic loops.
- **Adjacent sites** — for trunked `dvmhost` sites, `adj_site_map.yml` tells sites about each
  other so they can advertise adjacent-site broadcasts.

## Configuration files

The authoritative reference is the annotated
[`fne-config.example.yml`](https://github.com/DVMProject/dvmhost/blob/master/configs/fne-config.example.yml).
Related files in `configs/`:

| File | Purpose |
|------|---------|
| `fne-config.example.yml` | Main FNE configuration. |
| `talkgroup_rules.example.yml` | Talkgroup definitions, ACLs, routing. |
| `rid_acl.example.dat` | CSV of radio IDs allowed on the network. |
| `peer_list.example.dat` | CSV of peers (hosts, bridges, and linked FNEs) allowed to connect. |
| `adj_site_map.example.yml` | Adjacent-site mappings for trunked sites. |
| `key-container.ekc` | Cryptographic material for network-wide key loading (no example provided). |

### Main config sections

| Section | Controls |
|---------|----------|
| `master` | Listener `address` / `port` / `password` (`port + 1` carries metadata), `peerId`, worker count, connection limit, spanning tree, optional link `encrypted` / `presharedKey`, per-mode traffic permissions (`allowDMRTraffic`, `allowP25Traffic`, …), parrot behavior, P25 OTAR/KMF services, InfluxDB metrics, HA, and file paths for `talkgroup_rules` / `adj_site_map` / `crypto_container`. |
| `peers` | Upstream FNE peer-link connections (name, `masterAddress` / `masterPort` / `password`, `peerId`, optional encryption, location). |
| `system` | `identity`, peer ping timing, ACL update times, REST API listener, `radio_id` and `peer_acl` file paths. |
| `vtun` | Optional virtual network tunnel / P25 SNDCP dynamic IP allocation. |

Most parameters have sensible defaults; the ones you **must** review before first start are
the file paths for the ACL and rules files, the `master` password, and the `peerId`.

## Running `dvmfne`

```
usage: ./dvmfne [-vhf] [-p] [--syslog] [-c <configuration file>]

  -v        show version information
  -h        show help
  -f        foreground mode
  -p        promiscuous hub mode (repeat all traffic to all peers)
  --syslog  force logging to syslog
  -c <file> configuration file to use
```

```bash
/opt/dvm/bin/dvmfne -c /opt/dvm/fne-config.yml
```

Set `iAgreeNotToBeStupid: true` in the config to allow it to start.

## Jitter buffer

The FNE has an adaptive jitter buffer (`master.jitterBuffer`, typically tuned per peer). Sizing
and behavior are documented in
[`docs/FNE Jitter Buffer Configuration.md`](https://github.com/DVMProject/dvmhost/blob/master/docs/FNE%20Jitter%20Buffer%20Configuration.md).

## Provisioning and monitoring

- **REST API** — enable `system.restEnable` (behind a reverse proxy) for remote control and
  provisioning. Documented in `docs/TN.1100 - FNE REST API Documentation.md`.
- [`dvmprov`](../dvmprov.md) — a web front-end for the FNE REST API (archived).
- [`sysview`](supplementary.md) — near-realtime TUI monitor for a running FNE.
- [`tged`](supplementary.md) / [`peered`](supplementary.md) — TUI editors for the talkgroup
  rules and peer list files.

## Source / further reading

- Annotated config: [`fne-config.example.yml`](https://github.com/DVMProject/dvmhost/blob/master/configs/fne-config.example.yml)
- Network stack: `docs/TN.1000 - Network Stack Technical Documentation.md`
- FNE REST API: `docs/TN.1100 - FNE REST API Documentation.md`
- `fnecore` library: <https://github.com/DVMProject/fnecore>
