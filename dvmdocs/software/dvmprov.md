# DVMProv Web-Based FNE Provisioning Tool

**DVM Provisioning Manager** (`dvmprov`) is a Flask web application for managing a DVM FNE
(historically the "Converged FNE" / CFNE) through its REST API — editing peers, talkgroup
rules, and radio-ID ACLs from a browser instead of hand-editing YAML.

```{warning}
The [`dvmprov` repository](https://github.com/DVMProject/dvmprov) is **archived**. It targets an
older FNE REST API and may not track the current [`dvmfne`](dvmhost/dvmfne.md). Treat it as a
reference; verify against your FNE version before relying on it.
```

## Installation

```bash
git clone https://github.com/DVMProject/dvmprov
cd dvmprov
python3 -m venv .
source bin/activate
pip install -r requirements.txt
```

## Configuration

Copy `rest.example.py` to `rest.py` and point it at your FNE's REST API:

```python
rest_api_address = "127.0.0.1"
rest_api_port = 9990
rest_api_password = "PASSWORD"
```

The host running `dvmprov` must be able to reach the FNE REST API directly.

```{warning}
`dvmprov` can directly modify FNE configuration and **has no authentication of its own**. Keep
both it and the FNE REST API behind a reverse proxy (nginx or Apache) with authentication and
TLS.
```

## Running

```bash
(dvmprov)$ python dvmprov.py
```

It serves on `http://127.0.0.1:8180` by default.

| Argument | Description |
|----------|-------------|
| `-v` | Enable debug logging. |
| `-r` | Enable reverse-proxy support (required when accessed through a reverse proxy). |

For reverse-proxy setup, follow the Flask deployment guides for
[nginx](https://flask.palletsprojects.com/en/3.0.x/deploying/nginx/) or
[Apache HTTPD](https://flask.palletsprojects.com/en/3.0.x/deploying/apache-httpd/).

## Source

<https://github.com/DVMProject/dvmprov> (archived)
