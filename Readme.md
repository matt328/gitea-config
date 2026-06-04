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
git clone <repo> && cd provision-git
```

2. Optionally update variables:

- Edit `roles/gitea/templates/gitea.container.j2` to set `GITEA__server__ROOT_URL` and `SSH_DOMAIN` to your domain(s)
- If you want a different backup location or retention, edit `roles/gitea/templates/gitea-backup.sh.j2`

3. Run the playbook locally (it targets `localhost`):

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
2. Clone this repository and update `gitea.container.j2` if domain or ports differ.
3. Ensure the backup volume from the previous host is attached and mounted at `/mnt/backups`, or copy the latest
   `gitea-dump-*.zip` onto the new host's `/mnt/backups`.
4. Run the playbook to create directories and start services:

```bash
sudo ansible-playbook -i inventory.ini site.yml
```

5. Stop the Gitea service before restoring:

```bash
sudo systemctl stop gitea
```

6. Copy the desired backup archive into the Gitea data path the playbook expects, for example:

```bash
sudo cp /mnt/backups/gitea-dump-YYYY-MM-DD-XXXX.zip /var/lib/gitea/gitea/gitea-dump.zip
```

7. Inside the running or restored container, perform the restore (recommended to test on staging):

```bash
# If container is running as 'gitea' use podman exec
sudo podman exec -u git gitea /usr/local/bin/gitea restore -c /data/gitea/conf/app.ini -f /data/gitea/gitea-dump.zip
```

Note: Validate the restore command and options against the Gitea version you use. If unsure, run:

```bash
podman exec -it gitea /bin/sh
# then inspect: /usr/local/bin/gitea --help
```

8. Start the Gitea service if stopped:

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

## Customization and notes

- The `gitea.container.j2` template sets Gitea environment entries (`ROOT_URL`, `SSH_DOMAIN`, `SSH_PORT`). Change those
  to match your DNS and firewall.
- The playbook uses the Ansible podman collection to deploy NPM. If running on a non-DNF host, install podman and adjust
  `roles/base/tasks/main.yml` accordingly.
- Quadlet files are written to `/etc/containers/systemd`, so Gitea runs as a systemd-managed podman container: manage it
  with `systemctl` (system scope) or podman commands.
- Test restores on a staging host before relying on them for production DR procedures.

## Support

Potential improvements:

- Make domain, ports, backup path and retention configurable via `group_vars/all.yml`
- Add secure secrets management for database credentials (if you switch from built-in DB to an external DB)

## Contributing

Open a PR with improvements or suggestions. If possible, describe the expected DR behavior and desired backup retention
limits.
