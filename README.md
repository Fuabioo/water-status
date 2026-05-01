# water-status

Tiny daemon that watches the AYA (Costa Rica) public service-interruption API
for one or more configured locations and pushes Telegram-channel alerts when
an outage appears, changes (e.g. AYA extends the end-time), or clears.

Single static Go binary. Runs happily on a Raspberry Pi 3/4. SQLite for state.
Embedded web UI for configuration. No source published — only binaries.

## Install

```bash
brew install Fuabioo/tap/water-status
brew services start water-status
```

The first run creates `~/.config/water-status/config.yaml` with defaults and
opens the web UI on http://localhost:8080.

## How it works

- Calls `apigat.aya.go.cr/sitio/api/SitioWeb/Interrupciones` (the JSON API the
  official site uses) — no HTML scraping, far more stable.
- One worker per configured location, with jittered intervals and a
  process-wide minimum spacing between outbound requests. Default cadence is
  one request per location every 30m ± 20%, never closer than 5s apart. This
  is well below any reasonable rate-limit and avoids fingerprinting.
- Each outage is hashed by `(description, start, end)`. When AYA changes any
  of those, the channel gets an "ACTUALIZACIÓN" notice — the "outage
  permanency" requirement.
- A `schema_drift` flag is set when AYA returns an unparseable shape. The web
  UI shows a banner and the audit log records it; we deliberately stay quiet
  on the channel during AYA-side incidents to avoid flapping noise.

## Telegram

- **Channel** (`channel_id`) gets every outage state change: new, changed,
  cleared.
- **Chat** (`chat_id`) is where the bot answers `/check`, `/status`, `/help`.
  `/check` triggers an out-of-band poll. If outages are found, the channel
  gets the alert and the chat gets `"Checked. Outages found — see channel."`
  Otherwise the chat gets `"Checked. No outages."` and the channel stays
  silent.

For the bot to receive `/check`, the chat must be a private message or a
group where the bot is added. Group privacy mode must be off (set with
`/setprivacy` in @BotFather) for the bot to see commands.

## Build

Uses [`just`](https://github.com/casey/just) as the task runner.

```bash
just                    # native build (alias for `just build`)
just linux-armv7        # Raspberry Pi 3/4 (32-bit OS)
just linux-arm64        # Raspberry Pi 4/5 (64-bit OS)
just release            # all targets, optionally UPX-compressed, sha256'd
just --list             # show every recipe
just version-info       # print the embedded version trio
```

Override the version that gets baked in:

```bash
VERSION=v0.1.0 just release
```

Pure-Go — no CGO, cross-compile is a single env-var flip. Stripped binaries
are ~11MB; with UPX `--best --lzma` they drop to ~3MB.

## Run as a systemd service

Two units ship under `packaging/`:

- **`water-status.service`** — system-wide; runs as a dedicated
  `water-status` user with sandboxed filesystem access. Use this on a
  shared box.
- **`water-status.user.service`** — per-user; no root needed. Use this on a
  personal Pi where you log in as your own user.

### System install

```bash
# 1. Create a dedicated user with no shell.
sudo useradd --system --home-dir /var/lib/water-status \
             --create-home --shell /usr/sbin/nologin water-status

# 2. Drop config + binary in standard locations.
sudo install -d -o water-status -g water-status -m 750 /etc/water-status
sudo install -o water-status -g water-status -m 640 \
             config.example.yaml /etc/water-status/config.yaml
sudo install -m 755 water-status /usr/local/bin/water-status

# 3. Install + start the unit.
sudo install -m 644 packaging/water-status.service \
             /etc/systemd/system/water-status.service
sudo systemctl daemon-reload
sudo systemctl enable --now water-status

# 4. Tail logs.
journalctl -u water-status -f
```

The unit applies sensible hardening defaults: `NoNewPrivileges`,
`ProtectSystem=strict`, `ProtectHome`, `PrivateTmp`, `PrivateDevices`, plus
a `MemoryMax=128M` / `TasksMax=32` ceiling so any runaway becomes a
crash you'll see, not a slow ramp.

### Per-user install (no sudo)

```bash
mkdir -p ~/.local/bin ~/.config/systemd/user
cp water-status ~/.local/bin/
cp packaging/water-status.user.service \
   ~/.config/systemd/user/water-status.service
systemctl --user daemon-reload
systemctl --user enable --now water-status

# Keep the service running after you log out:
loginctl enable-linger "$USER"

# Tail logs:
journalctl --user -u water-status -f
```

### Updating

```bash
sudo systemctl stop water-status
sudo install -m 755 water-status /usr/local/bin/water-status
sudo systemctl start water-status
```

The state database (`/var/lib/water-status/state.db`) and the config
(`/etc/water-status/config.yaml`) are preserved across upgrades.

## Repo layout

This source repo lives at `gitea.fabiomora.dev:Fuabioo/water-status` (private).
The public mirror at `github.com/Fuabioo/water-status` only carries built
artifacts under `dist/` and the release workflow. `scripts/publish-release.sh`
ties the two together and bumps the Homebrew tap.

## Endpoints

| Method | Path                | Purpose                                     |
| ------ | ------------------- | ------------------------------------------- |
| GET    | `/`                 | Web UI                                      |
| GET    | `/api/health`       | liveness + schema-drift flag                |
| GET    | `/api/config`       | current YAML as JSON                        |
| PUT    | `/api/config`       | replace config                              |
| GET    | `/api/locations`    | proxy to AYA Ubicaciones (provinces, etc.)  |
| GET    | `/api/outages`      | active outages from local DB                |
| GET    | `/api/checks`       | recent audit log                            |
| POST   | `/api/check-now`    | force a poll across all locations           |
