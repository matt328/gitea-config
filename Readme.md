# Provisioning Gitea + Nginx Proxy Manager (podman)

## Overview

This repository contains an Ansible playbook and Quadlet/podman provisioning to run a self-hosted Gitea container and an
Nginx Proxy Manager (NPM) container. It is designed for a small DR-capable host (local podman + systemd quadlets).

Key components

- `site.yml` / `inventory.ini` — Ansible entrypoint (runs locally)
- `roles/base` — installs podman, cockpit and defines a homelab podman network quadlet
- `roles/gitea` — deploys Gitea as a containers/systemd quadlet, backup script and systemd timer
- Templates include: `gitea.container` (Quadlet), `gitea-backup.sh`, `gitea-backup.service` and `gitea-backup.timer`

## Important volumes and paths

- Gitea data: `/var/lib/gitea` (mounted into container as `/data`)
- NPM data: `/var/lib/npm/data`
- Let's Encrypt (NPM): `/var/lib/npm/letsencrypt`
- Backup destination (used by `gitea-backup.sh`): `/mnt/backups` (ensure this is a mounted backup disk)

## Prerequisites

- Linux host with podman and systemd (the playbook installs podman via DNF)
- Ansible (to run the playbook locally): `ansible-core` or `ansible`
- A dedicated backup volume mounted at `/mnt/backups` (or edit the backup script)
- Proper firewall & DNS entries for the domains you will use

## Quick install (fresh host)

1. Clone this repository onto the host:

```bash
git clone <repo> && cd gitea-config
```

2. Create your local environment configuration:

```bash
cp group_vars/all.local.example.yaml group_vars/all.local.yaml
```

3. Edit `group_vars/all.local.yaml` with your environment-specific values:

- `gitea_root_url` — your Gitea domain (e.g., `https://git.example.com/`)
- `gitea_ssh_domain` — SSH domain for Gitea (e.g., `git.example.com`)
- `backup_dir` — path to your backup volume (e.g., `/mnt/backups`)
- `backup_limit_gb` — maximum size for backups before older ones are deleted
- Other settings as needed

4. Run the playbook locally (it targets `localhost`):

```bash
sudo ansible-playbook -i inventory.ini site.yml
```

## What the playbook does

- Installs required system packages (`podman`, `cockpit`, `cockpit-podman`)
- Deploys a Podman network quadlet at `/etc/containers/systemd/homelab_net.network`
- Creates `/var/lib/gitea` and NPM directories
- Writes a containers/systemd quadlet file for Gitea (`/etc/containers/systemd/gitea.container`)
- Starts the Gitea systemd scope/service (quadlet-managed container)
- Deploys a backup script and systemd oneshot + timer to run daily at 02:00
- Deploys Nginx Proxy Manager container via the Ansible podman module

## Backups and DR (Disaster Recovery)

### Design notes

- The backup process performs a Gitea "dump" inside the Gitea container and moves the generated archive to `$BACKUP_DIR`
  (defaults to `/mnt/backups`).
- The script keeps total backup usage below `LIMIT_GB` (default `20GB`). Adjust as needed.

### Recovery workflow (new host or after disk replacement)

1. Provision a new host (same OS family) and install podman + systemd + ansible (or run the playbook steps manually).
2. Clone this repository and create your local configuration:

```bash
cp group_vars/all.local.example.yaml group_vars/all.local.yaml
```

3. Edit `group_vars/all.local.yaml` if domain, ports, or backup paths differ from your previous setup.
4. Ensure the backup volume from the previous host is attached and mounted at your configured `backup_dir`, or copy the latest
   `gitea-dump-*.zip` onto the new host's backup directory.
5. Run the playbook to create directories and start services:

```bash
sudo ansible-playbook -i inventory.ini site.yml
```

6. Stop the Gitea service before restoring:

```bash
sudo systemctl stop gitea
```

7. Copy the desired backup archive into the Gitea data path the playbook expects, for example:

```bash
sudo cp /mnt/backups/gitea-dump-YYYY-MM-DD-XXXX.zip /var/lib/gitea/gitea/gitea-dump.zip
```

8. Inside the running or restored container, perform the restore (recommended to test on staging):

```bash
# If container is running as 'gitea' use podman exec
sudo podman exec -u git gitea /usr/local/bin/gitea restore -c /data/gitea/conf/app.ini -f /data/gitea/gitea-dump.zip
```

Note: Validate the restore command and options against the Gitea version you use. If unsure, run:

```bash
podman exec -it gitea /bin/sh
# then inspect: /usr/local/bin/gitea --help
```

9. Start the Gitea service if stopped:

```bash
sudo systemctl start gitea
```

## Verification

- Visit the Gitea UI at the `ROOT_URL` configured in `gitea.container.j2` (port `3000` in the quadlet)
- Verify repositories, users, and webhooks
- Check the backup timer status:

```bash
sudo systemctl status gitea-backup.timer
```

- Spot-check `/mnt/backups` for newly created dumps

## Configuration

This repository uses an Ansible pattern for managing environment-specific configuration:

### Default Configuration (`group_vars/all.yaml`)
Contains sensible defaults that work for most setups. **Do NOT commit local changes here.**

### Local Configuration (`group_vars/all.local.yaml`)
Your environment-specific values go here. **This file is gitignored and should never be committed.**

To create your local config:
```bash
cp group_vars/all.local.example.yaml group_vars/all.local.yaml
```

Then edit `group_vars/all.local.yaml` with your values. Ansible will automatically merge these variables with the defaults, allowing you to override only what you need.

### Available Configuration Options
See `group_vars/all.local.example.yaml` for a complete list of configurable options with descriptions.

## Customization and notes

- The local configuration file (`group_vars/all.local.yaml`) overrides defaults in `group_vars/all.yaml`
- The playbook uses the Ansible podman collection to deploy NPM. If running on a non-DNF host, install podman and adjust
  `roles/base/tasks/main.yml` accordingly.
- Quadlet files are written to `/etc/containers/systemd`, so Gitea runs as a systemd-managed podman container: manage it
  with `systemctl` (system scope) or podman commands.
- Test restores on a staging host before relying on them for production DR procedures.

## Support

Potential future improvements:

- Add secure secrets management for database credentials (if you switch from built-in DB to an external DB)
- Add support for custom Gitea app.ini configuration via group_vars

## Contributing

Open a PR with improvements or suggestions. If possible, describe the expected DR behavior and desired backup retention
limits.
