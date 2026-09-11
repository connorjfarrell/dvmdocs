# `dvmcmd` Command-Line Interface

`dvmcmd` is a small utility that sends remote-control commands to a running
[`dvmhost`](dvmhost.md) or [`dvmfne`](dvmfne.md) instance through its **REST API**. It is
useful for scripting, cron jobs, and quick operational changes without editing config or
restarting the service.

## Prerequisites

The target instance must have the REST API enabled and reachable:

- `dvmhost`: `network.restEnable: true` with `restAddress` / `restPort` / `restPassword`
  (and optionally `restSsl`).
- `dvmfne`: `system.restEnable: true` with the same parameters.

```{warning}
Do not expose the REST API port directly to the internet. Bind it to localhost or a trusted
network, or proxy it through nginx / Apache with TLS.
```

## Usage

```
usage: ./dvmcmd [-dvhs] [-a <address>] [-p <port>] [-P <password>] <command> <arguments ...>

  -d        enable debug
  -v        show version information
  -h        show help
  -a        remote address
  -p        remote port
  -P        remote authentication password
  -s        use HTTPS/SSL
```

Run `dvmcmd -h` for the full list of commands and their arguments. Example:

```bash
dvmcmd -a 127.0.0.1 -p 9990 -P 'PASSWORD' <command> <args>
```

## Related tooling

- [`pydvm`](https://github.com/DVMProject/pydvm) — a Python library for interacting with DVM
  runtimes over the same REST API, for more involved automation.
- Full REST API reference: `docs/TN.1101 - DVMHost REST API Documentation.md` and
  `docs/TN.1100 - FNE REST API Documentation.md` in the `dvmhost` repository.
