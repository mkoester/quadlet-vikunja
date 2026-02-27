# quadlet-vikunja

Quadlet setup for [Vikunja](https://vikunja.io) — the to-do app to organize your life (`docker.io/vikunja/vikunja:latest`).

This project was created with the help of Claude Code and https://github.com/mkoester/quadlet-my-guidelines/blob/main/new_quadlet_with_ai_assistance.md.

## Files in this repo

| File | Description |
|---|---|
| `vikunja.container` | Quadlet unit file |
| `vikunja.env` | Default environment variables |
| `vikunja.override.env.template` | Template for local overrides (public URL, JWT secret) |
| `vikunja-backup.service` | Systemd service: backs up SQLite DB and uploaded files |
| `vikunja-backup.timer` | Systemd timer: triggers the backup daily |

## Setup

```sh
REPO_URL=https://github.com/mkoester/quadlet-vikunja.git
REPO=~vikunja/quadlet-vikunja
```

```sh
# 1. Create service user (regular user, home in /var/lib)
sudo useradd -m -d /var/lib/vikunja -s /usr/sbin/nologin vikunja

# 2. Enable linger
sudo loginctl enable-linger vikunja

# 3. Clone this repo into the service user's home
sudo -u vikunja git clone $REPO_URL $REPO

# 4. Create quadlet and data directories
sudo -u vikunja mkdir -p ~vikunja/.config/containers/systemd
sudo -u vikunja mkdir -p ~vikunja/{db,files}

# 5. Create .override.env from template and fill in required values
sudo -u vikunja cp $REPO/vikunja.override.env.template $REPO/vikunja.override.env
sudo -u vikunja nano $REPO/vikunja.override.env

# 6. Symlink all quadlet files from the repo
sudo -u vikunja ln -s $REPO/vikunja.container ~vikunja/.config/containers/systemd/vikunja.container
sudo -u vikunja ln -s $REPO/vikunja.env ~vikunja/.config/containers/systemd/vikunja.env
sudo -u vikunja ln -s $REPO/vikunja.override.env ~vikunja/.config/containers/systemd/vikunja.override.env

# 7. Reload and start
sudo -u vikunja XDG_RUNTIME_DIR=/run/user/$(id -u vikunja) systemctl --user daemon-reload
sudo -u vikunja XDG_RUNTIME_DIR=/run/user/$(id -u vikunja) systemctl --user start vikunja

# 8. Verify
sudo -u vikunja XDG_RUNTIME_DIR=/run/user/$(id -u vikunja) systemctl --user status vikunja
```

## Configuration

`vikunja.env` contains non-sensitive defaults:

| Variable | Default | Description |
|---|---|---|
| `VIKUNJA_DATABASE_TYPE` | `sqlite` | Database backend |
| `VIKUNJA_SERVICE_ENABLEREGISTRATION` | `true` | Allow new user sign-ups |

`vikunja.override.env` (created from the template) holds instance-specific and sensitive values:

| Variable | Description |
|---|---|
| `VIKUNJA_SERVICE_PUBLICURL` | Full public URL of your instance, e.g. `https://vikunja.example.com/` |
| `VIKUNJA_SERVICE_JWTSECRET` | Secret key for JWT tokens — generate once with `openssl rand -hex 32` |

Without `VIKUNJA_SERVICE_JWTSECRET`, a new secret is generated on every container restart, invalidating all active sessions.

## Reverse proxy (Caddy)

Add the following snippet to your Caddyfile:

```
@vikunja host vikunja.my_domain.tld
handle @vikunja {
    reverse_proxy localhost:3456
}
```

And add a DNS A/CNAME record for `vikunja.my_domain.tld` pointing to your server.

## Backup

The backup writes a consistent SQLite snapshot and an incremental copy of the files directory to `/var/backups/vikunja/`. Requires `sqlite3` and `rsync` to be installed on the host (`sudo dnf install sqlite rsync` / `sudo apt install sqlite3 rsync`). A remote machine pulls via `rsync` over SSH using the shared `backupuser`. See the [general backup setup](https://github.com/mkoester/quadlet-my-guidelines#backup) for the one-time server-wide setup (group, backup user, SSH key).

```sh
# 1. Create backup staging directories (owned by vikunja, readable by backup-readers group)
sudo mkdir -p /var/backups/vikunja/files
sudo chown -R vikunja:backup-readers /var/backups/vikunja
sudo chmod 750 /var/backups/vikunja

# 2. Symlink the backup service and timer from the repo
sudo -u vikunja mkdir -p ~vikunja/.config/systemd/user
sudo -u vikunja ln -s $REPO/vikunja-backup.service ~vikunja/.config/systemd/user/vikunja-backup.service
sudo -u vikunja ln -s $REPO/vikunja-backup.timer ~vikunja/.config/systemd/user/vikunja-backup.timer

# 3. Enable and start the timer
sudo -u vikunja XDG_RUNTIME_DIR=/run/user/$(id -u vikunja) systemctl --user daemon-reload
sudo -u vikunja XDG_RUNTIME_DIR=/run/user/$(id -u vikunja) systemctl --user enable --now vikunja-backup.timer
```

### On the remote (backup) machine

```sh
rsync -az backupuser@vikunja-host:/var/backups/vikunja/ /path/to/local/backup/vikunja/
```

## Notes

- Port `3456` is bound to `127.0.0.1` only — place a reverse proxy in front for external access.
- The SQLite database is stored at `~vikunja/db/vikunja.db` on the host.
- Uploaded file attachments are stored at `~vikunja/files/` on the host.
- The image is built `FROM scratch` (Go binary only) — no health check is possible from inside the container.
- `AutoUpdate=registry` is enabled; activate the timer once to get automatic image updates:
  ```sh
  sudo -u vikunja XDG_RUNTIME_DIR=/run/user/$(id -u vikunja) systemctl --user enable --now podman-auto-update.timer
  ```
- To prune old images automatically, enable the system-wide prune timer (see [image pruning setup](https://github.com/mkoester/quadlet-my-guidelines#image-pruning) for the one-time system setup). Replace `30` with the desired retention period in days:
  ```sh
  sudo -u vikunja XDG_RUNTIME_DIR=/run/user/$(id -u vikunja) systemctl --user enable --now podman-image-prune@30.timer
  ```
