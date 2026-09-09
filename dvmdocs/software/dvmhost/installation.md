# Installation

`dvmhost` and its companion applications are built from a single CMake project. These
instructions summarize the upstream
[`dvmhost` README](https://github.com/DVMProject/dvmhost/blob/master/README.md), which is the
authoritative source.

## Dependencies

On Debian / Ubuntu / Raspberry Pi OS:

```bash
sudo apt-get install build-essential cmake git libasio-dev libncurses-dev libssl-dev
```

- **ASIO** — header-only networking library. If you install it manually instead of from a
  package, pass `-DWITH_ASIO=/path/to/asio` to CMake.
- **ncurses** — TUI support.
- **OpenSSL** — REST API TLS and cryptographic material.
- Optional: `libdw-dev` (build) and `elfutils` (runtime) for detailed crash stack traces.

## Build

```bash
git clone https://github.com/DVMProject/dvmhost.git
cd dvmhost
mkdir build && cd build
cmake [options] ..
make
```

### Common CMake options

| Option | Effect |
|--------|--------|
| `-DCROSS_COMPILE_ARM=1` | Cross-compile for generic ARM 32-bit (32-bit Raspberry Pi OS, Bullseye+). |
| `-DCROSS_COMPILE_AARCH64=1` | Cross-compile for ARM64 (Raspberry Pi 4+, other ARM64). |
| `-DWITH_ASIO=/path/to/asio` | Use a manually extracted ASIO (required for ARM 32-bit cross-compile). |
| `-DENABLE_SETUP_TUI=0` | Disable the setup / calibration TUI. |
| `-DENABLE_TUI_SUPPORT=0` | Disable TUI support project-wide (also drops `dvmmon`, `sysview`, `tged`, `peered`). |

### Cross-compilation toolchains

- **ARM 32-bit:** `arm-linux-gnueabihf-gcc`, `arm-linux-gnueabihf-g++`
- **ARM 64-bit:** `aarch64-linux-gnu-gcc`, `aarch64-linux-gnu-g++`

Install the matching architecture libraries, e.g. for ARM64:

```bash
sudo dpkg --add-architecture arm64
sudo apt-get update
sudo apt-get install libasio-dev:arm64 libncurses-dev:arm64 libssl-dev:arm64
```

```{note}
Support for cross-compiling for the Raspberry Pi 1/2/3 was removed in DVM R05A02. Use a
Raspberry Pi 4+ or an x86_64 host.
```

## Install

Running DVM out of the `build` folder is **not** recommended. Choose one:

### Tarball (installs under `/opt`)

```bash
make strip
make tarball
sudo tar xvzf dvmhost_R04Gxx_<arch>.tar.gz -C /opt
```

### Legacy install (`/opt/dvm`)

```bash
make strip
sudo make old_install
sudo make old_install-service   # optional: install the systemd service
```

`make strip` before any install/tarball step reduces binary size.

## Post-install notes

- **Config migration:** `tools/config_annotator.py` compares an existing `config.yml` against
  the example and re-adds missing parameters / removes invalid ones. Back up your config first.
  It only works on the `dvmhost` config file.
- **Socket buffers:** ensure `net.core.rmem_max` and `net.core.wmem_max` are at least
  `524288`, otherwise you'll see "Could not resize socket recv/send buffer" warnings.
- **REST API:** do not expose the REST API port directly to the internet — proxy it through
  nginx or similar.

## Raspberry Pi preparation

On Raspberry Pi OS / Debian you must free the GPIO serial port before you can flash or talk to
an STM32 modem/hotspot over UART:

```bash
sudo systemctl disable bluetooth.service serial-getty@ttyAMA0.service
sudo systemctl mask serial-getty@ttyAMA0.service
grep '^dtoverlay=disable-bt' /boot/config.txt || echo 'dtoverlay=disable-bt' | sudo tee -a /boot/config.txt
sudo sed -i 's/^console=serial0,115200 *//' /boot/cmdline.txt
```

Reboot afterward. (On Bookworm the `serial-getty@ttyAMA0` unit is regenerated on boot unless
masked, hence the `mask` step.)

## Source

<https://github.com/DVMProject/dvmhost/blob/master/README.md>
