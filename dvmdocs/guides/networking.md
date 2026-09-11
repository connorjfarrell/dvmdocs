# FNE Networking & Topology

This guide explains how DVM instances connect through
[Fixed Network Equipment (`dvmfne`)](../software/dvmhost/dvmfne.md) and how traffic is routed.

## Roles

```{mermaid}
graph TD
    subgraph SiteA["Site A (trunked)"]
      HA1["dvmhost (CC)"]
      HA2["dvmhost (VC)"]
    end
    HB["dvmhost (Site B, conventional)"]
    BR["dvmbridge (AllStar / analog)"]
    CON["Desktop Dispatch Console"]
    FNE1["dvmfne 1"]
    FNE2["dvmfne 2"]
    HC["dvmhost (Site C)"]

    HA1 -->|peer| FNE1
    HA2 -->|peer| FNE1
    HB -->|peer| FNE1
    BR -->|peer| FNE1
    CON -->|peer| FNE1
    FNE1 <-->|peer link| FNE2
    FNE2 -->|peer| HC
```

- **Downstream peer** — a `dvmhost`, `dvmbridge`, `dvmpatch`, or console that connects *to* an
  FNE. Each has a unique {term}`Peer ID` (`network.id`).
- **Master** — the FNE's listener that downstream peers connect to (`master` block:
  `address`, `port`, `password`, `peerId`).
- **Peer link** — a connection *between* two FNEs, configured in the `peers:` block of one of
  them. Lets separate networks exchange traffic.

## Ports

| Purpose | Default | Notes |
|---------|---------|-------|
| FNE master (peer traffic) | `62031/udp` | `port + 1` (`62032`) carries metadata — open both. |
| Host ↔ CC/VC RPC | `9890/tcp` | Internal trunking coordination (`system.config.controlCh` / `voiceChNo`). |
| REST API | `9990/tcp` | Optional remote control / provisioning. Keep behind a reverse proxy. |
| P25 KMF / OTAR | `64414/udp` | Only if `master.kmfServicesEnabled`. |

```{warning}
Linux may cap socket buffers below what DVM needs. Ensure `net.core.rmem_max` and
`net.core.wmem_max` are at least `524288` or the FNE logs
"Could not resize socket recv/send buffer" on startup.
```

## Talkgroup routing

`talkgroup_rules.yml` on the FNE defines every valid talkgroup and how it routes. Key
`config` fields per `groupVoice` entry:

| Field | Effect |
|-------|--------|
| `active` | Talkgroup is enabled. |
| `affiliated` | Only repeat to sites that have an affiliated radio (trunking). |
| `inclusion` / `exclusion` | Explicit allow / deny lists of peer IDs. |
| `always` | Peer IDs that always receive this talkgroup regardless of affiliation. |
| `rewrite` | Per-peer TGID/slot rewrite (bridge TGID *X* on one peer to TGID *Y* on another). |
| `preferred` | Trunking: CC peer IDs preferred for access; non-preferred sites return a DENY, triggering roaming. |
| `rid_permitted` | Restrict transmit on this talkgroup to specific radio IDs. |
| `parrot` | Record-and-playback test talkgroup (see {term}`Parrot`). |

`source.tgid` is the talkgroup number; `source.slot` is the DMR timeslot.

## Access control

- **`rid_acl.dat`** — CSV of radio IDs permitted on the network. Enforced when
  `radio_id.acl: true` (host) or via the FNE.
- **`peer_list.dat`** — CSV of peer IDs permitted to connect to the FNE (downstream peers and
  linked FNEs). Enforced when `system.peer_acl.enable: true`.
- ACL and rules files can be **pushed from the FNE** to hosts. If you do that, set the host's
  local update `time` to `0` so it doesn't overwrite the pushed rules.

## Multi-FNE networks

- Each linked FNE is a `peers:` entry pointing at the other's master (`masterAddress` /
  `masterPort` / `password`, plus a unique `peerId`).
- **Spanning tree** (`master.enableSpanningTree: true`) prevents routing loops when FNEs are
  meshed — leave it enabled. `spanningTreeFastReconnect` speeds recovery after a link drop.
- `maskOutboundPeerID` hides internal peer IDs from public-facing links.
- **Promiscuous / hub mode** (`dvmfne -p`) repeats all traffic to all peers, ignoring
  affiliation — useful for a simple "everyone hears everything" hub, not for trunking.

## Trunked multi-site (adjacent sites)

For trunking that spans sites, `adj_site_map.yml` on the FNE lists each CC peer and its
`neighbors` (other CC peer IDs). Sites use this to broadcast adjacent-site information so
radios can roam between them.

```yaml
peers:
  - peerId: 1234567
    active: true
    neighbors: [1234568, 1234569]
```

Also review `disallowAdjStsBcast` / `disallowExtAdjStsBcast` / `allowConvSiteAffOverride` in
the FNE `master` block.

## Link encryption

Peer connections can be AES-256 encrypted end-to-end — see [Encryption](encryption.md).

## Jitter buffer

The FNE buffers inbound frames to smooth network jitter (`master.jitterBuffer`, usually tuned
per peer). Sizing guidance:
[`docs/FNE Jitter Buffer Configuration.md`](https://github.com/DVMProject/dvmhost/blob/master/docs/FNE%20Jitter%20Buffer%20Configuration.md).

## Source / further reading

- Network stack technical note: `docs/TN.1000` — <https://github.com/DVMProject/dvmhost/tree/master/docs>
- Annotated FNE config: <https://github.com/DVMProject/dvmhost/blob/master/configs/fne-config.example.yml>
- Talkgroup rules example: <https://github.com/DVMProject/dvmhost/blob/master/configs/talkgroup_rules.example.yml>
- `fnecore` library: <https://github.com/DVMProject/fnecore>
