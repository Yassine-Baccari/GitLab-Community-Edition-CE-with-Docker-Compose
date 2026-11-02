# GitLab Community Edition (CE) with Docker Compose

Self-hosted GitLab CE, run with Docker Compose, with all persistent data stored on the host under `./gitlab/` (no named volumes, no `$GITLAB_HOME`).

- Image: `gitlab/gitlab-ce` (pinned version, set in `.env`)
- Pinned version in this guide: `19.4.2-ce.0` (latest tag seen on Docker Hub in mid-October 2026). Check https://hub.docker.com/r/gitlab/gitlab-ce/tags for a newer patch of the same minor release before installing.
- Reference: https://docs.gitlab.com/install/docker/installation/

---

## Contents

1. [Directory structure](#1-directory-structure)
2. [Configuration files](#2-configuration-files)
3. [Installation](#3-installation)
4. [Verification and troubleshooting](#4-verification-and-troubleshooting)
5. [Persistence validation](#5-persistence-validation)
6. [Backup and restore](#6-backup-and-restore)
7. [Maintenance and upgrades](#7-maintenance-and-upgrades)

---

## 1. Directory structure

```text
gitlab-docker/
├── docker-compose.yml
├── .env
├── .gitignore
├── README.md
└── gitlab/
    ├── config/    -> /etc/gitlab       (gitlab.rb, gitlab-secrets.json, ssl/)
    ├── logs/      -> /var/log/gitlab   (all service logs)
    └── data/      -> /var/opt/gitlab   (repositories, database, uploads, backups/)
```

---

## 2. Configuration files

### `docker-compose.yml`

```yaml
services:
  gitlab:
    image: gitlab/gitlab-ce:${GITLAB_VERSION}
    container_name: gitlab-ce
    restart: always
    hostname: ${GITLAB_HOSTNAME}
    shm_size: 256m
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url '${GITLAB_EXTERNAL_URL}'
        gitlab_rails['gitlab_shell_ssh_port'] = ${GITLAB_SSH_PORT}
    ports:
      - "${GITLAB_HTTP_PORT}:${GITLAB_HTTP_PORT}"
      - "${GITLAB_HTTPS_PORT}:${GITLAB_HTTPS_PORT}"
      - "${GITLAB_SSH_PORT}:22"
    volumes:
      - ./gitlab/config:/etc/gitlab
      - ./gitlab/logs:/var/log/gitlab
      - ./gitlab/data:/var/opt/gitlab
```

Notes:

- Host and container ports are identical for HTTP and HTTPS because GitLab's bundled NGINX listens on the port written in `external_url`. With the defaults (80/443) this is automatic.
- SSH maps host `2222` to container `22`. `gitlab_shell_ssh_port` is the documented Omnibus setting that makes SSH clone URLs show port 2222.
- No secrets are stored in this file.
- Values in `gitlab/config/gitlab.rb` take precedence over `GITLAB_OMNIBUS_CONFIG`.

### `.env`

```dotenv
# ---- GitLab CE image (pinned; never use "latest") ----
GITLAB_VERSION=19.4.2-ce.0

# ---- Address users type in the browser ----
# Examples (pick ONE):
#   Localhost : http://localhost
#   LAN IP    : http://192.168.1.50
#   DNS name  : https://gitlab.example.com
GITLAB_EXTERNAL_URL=http://localhost

# Hostname only (no scheme, no port). Must match the host in the URL above.
GITLAB_HOSTNAME=localhost

# ---- Ports ----
GITLAB_HTTP_PORT=80
GITLAB_HTTPS_PORT=443
GITLAB_SSH_PORT=2222
```

### `.gitignore`

```gitignore
# GitLab runtime data: repos, DB, logs, secrets, backups
/gitlab/

# Local environment overrides
.env

# Backup archives wherever they end up
*.tar
*.tar.gz
*.tgz
```

To version your settings, copy `.env` to `.env.example` and commit that instead.

### Choosing the external URL

`GITLAB_EXTERNAL_URL` is the address people use in their browser. GitLab uses it for links, redirects and clone URLs, so it must be reachable from the client.

| Access method | `GITLAB_EXTERNAL_URL` | `GITLAB_HOSTNAME` | Notes |
|---|---|---|---|
| Localhost | `http://localhost` | `localhost` | Only works from the same machine. Fine for testing, not for production. |
| LAN IP | `http://192.168.1.50` | `192.168.1.50` | Use a static IP or DHCP reservation, otherwise links break when the IP changes. |
| DNS hostname | `https://gitlab.example.com` | `gitlab.example.com` | Best for production. Create the DNS A record first. |

With an `https://` URL:

- GitLab tries to obtain a free Let's Encrypt certificate automatically. This only works if the server is reachable from the public internet on ports 80 and 443.
- For private networks or your own certificates, place them in `./gitlab/config/ssl/` named after the hostname: `gitlab.example.com.crt` and `gitlab.example.com.key`.
- With a plain `http://` URL, nothing listens on 443, so the HTTPS port mapping is harmless but unused.

If you change the HTTP or HTTPS port, include it in the URL as well, for example `GITLAB_HTTP_PORT=8929` with `GITLAB_EXTERNAL_URL=http://192.168.1.50:8929`.

---

## 3. Installation

### Step 1: Prerequisites

```bash
docker --version          # Docker Engine installed (Docker Desktop is not required)
docker compose version    # Compose v2 plugin present
```

Recommended resources for a small installation:

| Resource | Minimum | Comfortable |
|---|---|---|
| CPU | 4 cores | 4-8 cores |
| RAM | 4 GB (plus ~2 GB swap) | 8 GB |
| Disk | 10 GB for the application | 50 GB+ SSD, sized to your repositories plus backups |

GitLab is memory-hungry. With less than 4 GB it is often OOM-killed or stuck restarting.

Check that the ports are free:

```bash
sudo ss -tulpn | grep -E ':(80|443|2222)\b' || echo "ports free"
```

### Step 2: Create directories

```bash
mkdir -p gitlab-docker && cd gitlab-docker
mkdir -p gitlab/config
mkdir -p gitlab/logs
mkdir -p gitlab/data
```

Permissions: the container initializes as root and creates and chowns the subdirectories it needs, so `chmod 777` is not required and should be avoided.

```bash
chmod 755 gitlab gitlab/logs gitlab/data
chmod 700 gitlab/config     # holds gitlab-secrets.json (encryption keys)
```

On hosts with **SELinux** (RHEL, Rocky, Fedora), add `:Z` to each volume, for example `./gitlab/config:/etc/gitlab:Z`.

Avoid network filesystems (NFS/SMB) for these directories; GitLab's ownership model often breaks on them.

### Step 3: Create the configuration files

Create `docker-compose.yml`, `.env` and `.gitignore` (section 2) inside `gitlab-docker/`, then edit `.env` to set your URL and hostname.

### Step 4: Start GitLab

```bash
docker compose config     # validate and print the file with .env values substituted
docker compose pull       # download the pinned image (~1.5 GB compressed)
docker compose up -d      # create and start the container in the background
```

Check that `docker compose config` shows your real URL and ports, and the `./gitlab/...` paths expanded to absolute paths.

### Step 5: Monitor initialization

```bash
docker compose logs -f gitlab     # Ctrl+C stops following; GitLab keeps running
docker compose ps                 # STATUS goes from "starting" to "healthy"
```

The first start typically takes 3-10 minutes (longer on slow disks). You will see Omnibus `reconfigure`, database migrations and services starting. Browser 502 errors during this time are normal.

### Step 6: Access GitLab

Open the URL you set in `GITLAB_EXTERNAL_URL`.

- Username: `root`
- Password: retrieve it with:

```bash
docker exec -it gitlab-ce grep 'Password:' /etc/gitlab/initial_root_password
```

Because `/etc/gitlab` is mapped to the host, you can also read it directly:

```bash
grep 'Password:' gitlab/config/initial_root_password
```

This file is deleted automatically 24 hours after the first container start, so log in and change the password before then.

**Change the root password:** click your avatar, choose **Edit profile**, then **Password**. Enter the current password and a new strong one, then save. Afterwards, enable two-factor authentication for root, and review **Admin > Settings > General > Sign-up restrictions** if the instance is exposed publicly.

---

## 4. Verification and troubleshooting

### Verification

```bash
# Container status (expect "Up ... (healthy)")
docker compose ps
docker inspect --format '{{.State.Status}} / {{.State.Health.Status}}' gitlab-ce

# GitLab internal services (expect "run:" for puma, sidekiq, gitaly, postgresql, redis, nginx...)
docker exec -it gitlab-ce gitlab-ctl status

# Full built-in health check (takes a minute or two)
docker exec -it gitlab-ce gitlab-rake gitlab:check

# HTTP accessibility from the host (200, or a 302 to /users/sign_in, is normal)
curl -I http://localhost
curl -s http://localhost/-/health       # "GitLab OK"
curl -s http://localhost/-/readiness    # JSON with per-service status

# Persistent volume mappings: host path -> container path
docker inspect gitlab-ce --format '{{range .Mounts}}{{.Type}}  {{.Source}} -> {{.Destination}}{{"\n"}}{{end}}'
```

The mounts output should list only `bind` entries pointing at your `gitlab-docker/gitlab/config`, `logs` and `data` directories, and no `volume` entries. If your external URL is not localhost, adjust the `curl` host. The health and readiness endpoints are reachable from localhost by default.

### Troubleshooting

**Container keeps restarting**

```bash
docker compose logs --tail 200 gitlab
docker inspect --format 'OOMKilled={{.State.OOMKilled}} ExitCode={{.State.ExitCode}}' gitlab-ce
```

`OOMKilled=true` means insufficient RAM. Other common causes: a syntax error in `GITLAB_OMNIBUS_CONFIG` (visible in the reconfigure output) and permission problems.

**Permission problems** (`Permission denied`, `chown` failures)

```bash
ls -ld gitlab gitlab/*
docker exec -it gitlab-ce gitlab-ctl reconfigure     # re-applies ownership on mounted dirs
```

Check for SELinux denials (`sudo ausearch -m avc -ts recent`) and add `:Z` to the mounts if needed. Do not use `chmod -R 777`.

**Port conflicts** (`bind: address already in use`)

```bash
sudo ss -tulpn | grep -E ':(80|443|2222)\b'
```

Stop the conflicting service (Apache, NGINX, another container) or change the ports in `.env`. If you change the HTTP/HTTPS port, include it in `GITLAB_EXTERNAL_URL`. Then run `docker compose up -d` again.

**GitLab initialization taking too long**

```bash
docker exec -it gitlab-ce gitlab-ctl tail      # live logs from all internal services
docker stats gitlab-ce --no-stream             # CPU / RAM usage
```

Ten minutes is normal on modest hardware. If CPU and disk are active, wait. If the container stays unhealthy for 15+ minutes with no activity, read the logs for the failing service.

**Insufficient RAM**

```bash
free -h
```

Add swap as a safety net (the real fix is 4 GB RAM minimum, ideally 8 GB):

```bash
sudo fallocate -l 4G /swapfile && sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
```

Add the swapfile to `/etc/fstab` to make it permanent.

**Web interface not accessible**

- Confirm `docker compose ps` shows the container as `healthy` (a 502 in the first minutes is normal).
- Test locally with `curl -I http://localhost`. If that works but remote access does not, check the host firewall and cloud security groups. Docker-published ports can bypass `ufw` rules.
- Make sure the URL in your browser matches `GITLAB_EXTERNAL_URL` exactly (scheme, hostname, port).
- For DNS names, confirm resolution with `getent hosts gitlab.example.com`.
- If you edited `gitlab/config/gitlab.rb`, remember its values take precedence over `GITLAB_OMNIBUS_CONFIG`.

---

## 5. Persistence validation

Confirm data is stored on the host:

```bash
ls -la gitlab/config    # gitlab.rb, gitlab-secrets.json, initial_root_password, ssl/ ...
ls -la gitlab/logs      # nginx/, puma/, sidekiq/, gitaly/, gitlab-rails/ ...
ls -la gitlab/data      # git-data/, postgresql/, redis/, gitlab-rails/, backups/ ...
du -sh gitlab/*         # sizes grow as you use GitLab
```

Prove it survives container recreation. First create a test project in the web UI, then:

```bash
docker compose down      # stops and REMOVES the container; host data is untouched
ls gitlab/data           # still there
docker compose up -d     # new container, same image, same bind-mounted data
docker compose logs -f gitlab
```

After a few minutes, all users, projects, repositories, settings and configuration are intact, because the host directories (not the container) hold all state.

**Do not use `docker compose down -v` as normal maintenance.** This setup has no named volumes, so it would do little here, but it is a dangerous habit that deletes data in setups that do use them. Your data can only be destroyed by deleting `./gitlab/` yourself.

---

## 6. Backup and restore

GitLab needs two separate backups, and you need both:

1. **Application backup** (database, repositories, uploads, CI data), made with `gitlab-backup`.
2. **Configuration** (`gitlab.rb`, `gitlab-secrets.json`, SSL certificates). These are not included in the application backup. Without `gitlab-secrets.json`, a restored instance cannot decrypt 2FA secrets, CI variables and other encrypted data.

### Create a backup

```bash
# Application data -> ./gitlab/data/backups/<timestamp>_<date>_<version>_gitlab_backup.tar
docker exec -t gitlab-ce gitlab-backup create

# Configuration, archived separately
sudo tar -czf gitlab-config-$(date +%F).tar.gz -C gitlab config
```

### Protect the backups

```bash
sudo chmod 600 gitlab-config-*.tar.gz gitlab/data/backups/*.tar
sudo ls -l gitlab/data/backups
```

- Copy backups to a different disk or machine (`rsync`, `scp`, object storage). A backup on the same disk does not survive disk failure.
- Store the config archive separately from the application backup, so one leak does not expose both data and keys.
- Encrypt before sending offsite, for example: `gpg --symmetric --cipher-algo AES256 gitlab-config-2026-11-02.tar.gz`
- Automate with cron (example: daily at 02:00; adjust the path):

  ```bash
  0 2 * * * docker exec -t gitlab-ce gitlab-backup create CRON=1
  ```

- To let GitLab delete old backups itself, set `gitlab_rails['backup_keep_time'] = 604800` (seconds, i.e. 7 days) in `gitlab/config/gitlab.rb`.

### Restore

The target GitLab must be the **same version and edition** as the one that created the backup.

```bash
# 1. Restore config and start a container of the SAME version (same .env)
sudo tar -xzf gitlab-config-2026-11-02.tar.gz -C gitlab     # recreates ./gitlab/config
docker compose up -d                                         # wait until healthy

# 2. Place the backup file in the backups directory, owned by git
sudo cp 1730500000_2026_11_02_19.4.2_gitlab_backup.tar gitlab/data/backups/
docker exec -it gitlab-ce chown git:git /var/opt/gitlab/backups/1730500000_2026_11_02_19.4.2_gitlab_backup.tar

# 3. Stop the services that write to the database
docker exec -it gitlab-ce gitlab-ctl stop puma
docker exec -it gitlab-ce gitlab-ctl stop sidekiq
docker exec -it gitlab-ce gitlab-ctl status       # puma and sidekiq should show "down"

# 4. Restore. BACKUP= is the file name WITHOUT the trailing _gitlab_backup.tar
docker exec -it gitlab-ce gitlab-backup restore BACKUP=1730500000_2026_11_02_19.4.2

# 5. Reconfigure, restart and verify
docker exec -it gitlab-ce gitlab-ctl reconfigure
docker exec -it gitlab-ce gitlab-ctl restart
docker exec -it gitlab-ce gitlab-rake gitlab:check SANITIZE=true
```

Answer `yes` to the confirmation prompts. Test the restore procedure on a spare machine before you actually need it.

---

## 7. Maintenance and upgrades

### Everyday commands

```bash
docker compose stop              # stop GitLab (container kept)
docker compose start             # start it again
docker compose restart           # restart the container (full GitLab restart, takes a few minutes)
docker exec -it gitlab-ce gitlab-ctl restart    # restart only GitLab's internal services

docker compose logs -f gitlab                   # container output (follow)
docker compose logs --tail 100 gitlab
docker exec -it gitlab-ce gitlab-ctl tail       # all internal service logs
ls gitlab/logs                                  # the same logs, directly on the host
```

### Check the current version

```bash
docker exec -it gitlab-ce cat /opt/gitlab/embedded/service/gitlab-rails/VERSION
docker inspect --format '{{.Config.Image}}' gitlab-ce
```

The version is also shown at `/help` in the web UI when logged in.

### Pin a specific version

Set `GITLAB_VERSION` in `.env` to an exact tag such as `19.4.2-ce.0` (the `-ce.0` suffix is part of the tag). Never use `latest`: a plain `docker compose pull` could jump versions and break your upgrade path.

### Upgrade safely

GitLab does not allow skipping arbitrary versions. Upgrades must follow the supported upgrade path, which often includes required stop versions. Work out your exact route with GitLab's Upgrade Path tool (https://gitlab-com.gitlab.io/support/toolbox/upgrade-path/) and read the release notes for each target version.

```bash
# 1. Back up first. Always.
docker exec -t gitlab-ce gitlab-backup create
sudo tar -czf gitlab-config-pre-upgrade-$(date +%F).tar.gz -C gitlab config

# 2. Make sure no background migrations are still running
#    (Admin > Monitoring > Background migrations), and wait for them to finish.

# 3. Edit .env: set GITLAB_VERSION to the NEXT version on the supported path.
#    Go one hop at a time and wait for "healthy" before the next hop.

# 4. Apply
docker compose pull
docker compose up -d
docker compose logs -f gitlab          # migrations run automatically; wait for "healthy"

# 5. Verify
docker exec -it gitlab-ce cat /opt/gitlab/embedded/service/gitlab-rails/VERSION
docker exec -it gitlab-ce gitlab-rake gitlab:check
```

Why back up first: database migrations are one-way. If an upgrade fails midway, simply switching back to the old image does not work; the only way back is restoring a backup taken on the old version, and a backup can only be restored on the same version that created it.
