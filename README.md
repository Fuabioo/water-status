# water-status

Tiny daemon that watches AyA (Costa Rica) for water-service interruptions
across one or more configured locations, and pushes Telegram-channel alerts
when an outage appears, changes (e.g. AyA extends the end-time), or clears.

This repository is a **binaries-only mirror**. Source lives in a private
upstream repository; only release artifacts are published here.

## Install

Via Homebrew:

```bash
brew install Fuabioo/tap/water-status
```

The post-install hook seeds a default config in `~/.config/water-status/`,
installs a per-user systemd unit (Linux), and tries to start the service.
If automatic start fails (no D-Bus session at install time, common over
plain SSH), the printed caveats walk you through the manual command.

## Run

After install, the daemon listens on `http://localhost:8080` for the web
UI. Service control:

```bash
systemctl --user status   water-status
systemctl --user restart  water-status
journalctl --user -u water-status -f
```

To survive logout (recommended on a Pi):

```bash
loginctl enable-linger $USER
```

A hardened system-wide unit (dedicated user, sandboxed filesystem,
`MemoryMax=128M`) is shipped at `$(brew --prefix water-status)/share/water-status/water-status.service`
if you'd rather run that.

## Update

```bash
brew update && brew upgrade water-status
```

State (`~/.local/share/water-status/state.db`) and config
(`~/.config/water-status/config.yaml`) are preserved across upgrades.

## Releases

GitHub releases under [/releases](../../releases) carry the prebuilt
tarballs (`water-status-linux-{amd64,arm64,armv7}.tar.gz`) plus
their sha256 checksums. Each tarball bundles the binary, both systemd
units (per-user and system-wide), and an example config.

## License

[MIT](LICENSE).
