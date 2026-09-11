# Encryption & Key Management

DVM supports two distinct kinds of encryption that are easy to confuse:

1. **Network link encryption** — protects the traffic *between* DVM instances on the wire.
2. **Air interface (voice) encryption** — the P25/DMR over-the-air voice privacy that radios
   themselves perform; DVM passes it through and can help manage the keys.

```{warning}
Encryption features do not change the [usage policy](../overview/introduction.md): DVM is for
amateur and educational use only. Amateur radio regulations in many countries prohibit
encrypted transmissions on amateur allocations — know your local rules.
```

## Network link encryption

Any peer connection (host → FNE, or FNE ↔ FNE) can be encrypted with a pre-shared
**AES-256** key.

```yaml
# both ends must match
network:            # (host)   or   master: / peers[]: (FNE)
  encrypted: true
  presharedKey: "000102030405060708090A0B0C0D0E0F000102030405060708090A0B0C0D0E0F"
```

- The key **must** be exactly 32 hex bytes (64 characters, `0-9 A-F`).
- Set it on both ends of every link you want protected. Mismatched keys = no connection.
- This is transport protection only; it is unrelated to whether the *voice* is encrypted.

## Link Layer Authentication (LLA)

P25 LLA authenticates a radio to a site before it is allowed to register, using an AES key:

```yaml
system:
  config:
    secure:
      key: "000102030405060708090A0B0C0D0E0F"   # 16 hex bytes / 32 chars
    # require LLA before a radio may register
p25:
  requireLLAForReg: true
```

## Air interface (voice) encryption

The radios encrypt and decrypt voice; DVM relays the encrypted payload and the key/algorithm
identifiers. For DVM to *originate or terminate* encrypted audio (parrot playback, console
monitoring, analog bridging) it needs the traffic encryption keys (TEKs).

### Where keys live

| Component | File | Purpose |
|-----------|------|---------|
| [Desktop Dispatch Console](../software/dvmconsole.md) | `keys.clear` | Plaintext key entries, each matched to a **Key ID** referenced in `codeplug.yml`. Lets the console monitor/transmit on encrypted talkgroups. |
| [`dvmfne`](../software/dvmhost/dvmfne.md) | `key_container.ekc` | Encrypted key container (`master.crypto_container`, unlocked with a password) providing key material network-wide for keyloading. |

```yaml
master:
  crypto_container:
    enable: true
    file: key_container.ekc
    password: "PASSWORD"
    time: 30            # minutes between reloads
```

### P25 OTAR / KMF

The FNE can act as a Key Management Facility for P25 Over-The-Air-Rekeying:

```yaml
master:
  kmfServicesEnabled: true
  kmfOtarPort: 64414
```

This lets a KMF client distribute and update TEKs to radios across the network. Open
`64414/udp` to the KMF client only.

```{note}
Supported voice-encryption algorithms and key formats evolve. Confirm current support (DES-OFB,
AES-256, ARC4/ADP, key ID / algorithm ID handling) against the running `dvmhost` version and
ask in the [Discord](https://discord.gg/3pBe8xgrEz) — the technical notes in
`dvmhost/docs` are the authoritative reference.
```

## Source / further reading

- Host config (`network.encrypted`, `system.config.secure`): <https://github.com/DVMProject/dvmhost/blob/master/configs/config.example.yml>
- FNE config (`crypto_container`, `kmf*`): <https://github.com/DVMProject/dvmhost/blob/master/configs/fne-config.example.yml>
- REST API (key endpoints): `docs/TN.1100` / `docs/TN.1101` — <https://github.com/DVMProject/dvmhost/tree/master/docs>
- Console key setup: <https://github.com/DVMProject/dvmconsole>
