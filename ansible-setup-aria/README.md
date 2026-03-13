# Aria — Ansible Setup (`ansible-setup-aria`)

> Ansible automation for **l-srv-inf-001 "Aria"** — a self-hosted, isolated homelab infrastructure server running Docker.
> Remote access is provided via TwinGate (zero-trust); no ports are forwarded on the router.

---

## Table of Contents

- [Aria — Ansible Setup (`ansible-setup-aria`)](#aria--ansible-setup-ansible-setup-aria)
  - [Table of Contents](#table-of-contents)
  - [Architecture Overview](#architecture-overview)
    - [Key Design Decisions](#key-design-decisions)
  - [Directory Structure](#directory-structure)
  - [Deployed Services](#deployed-services)
    - [Phase 1 — Foundation](#phase-1--foundation)
    - [Phase 1 (cont.) — Monitoring Stack](#phase-1-cont--monitoring-stack)
    - [Phase 2 — Operations](#phase-2--operations)
    - [Phase 3 — Polish](#phase-3--polish)
  - [Deployment Order](#deployment-order)
  - [Prerequisites](#prerequisites)
    - [On the Ansible controller (your machine)](#on-the-ansible-controller-your-machine)
    - [On Aria (the target server)](#on-aria-the-target-server)
  - [Secrets \& Vault](#secrets--vault)
    - [Secrets managed by vault](#secrets-managed-by-vault)
    - [Vault workflow](#vault-workflow)
  - [Running the Playbook](#running-the-playbook)
    - [What the playbook does](#what-the-playbook-does)
  - [Port Reference](#port-reference)
  - [Adding a New Service](#adding-a-new-service)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                   l-srv-inf-001 "Aria"                          │
│                   Ubuntu · Docker CE                            │
│                                                                 │
│  ┌───────────┐  ┌──────────────────────────────────────────┐   │
│  │ TwinGate  │  │           Monitoring Stack                │   │
│  │ Connector │  │  Prometheus · Grafana · Alertmanager      │   │
│  │ (no port) │  │  Node Exporter · cAdvisor · Loki          │   │
│  └───────────┘  │  Promtail · Uptime Kuma                   │   │
│       │         └──────────────────────────────────────────┘   │
│       │ zero-trust                    │ alerts                  │
│       │         ┌─────────────────────▼────────────────────┐   │
│       │         │              ntfy                         │   │
│       │         │  push notifications → mobile/desktop      │   │
│       │         └──────────────────────────────────────────┘   │
│       │                                                         │
│  ┌────▼─────────────────────────────────────────────────────┐  │
│  │                 Management Tools                          │  │
│  │  Portainer · Semaphore · Dockge · Watchtower             │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                  Applications                            │  │
│  │  Vaultwarden · Authelia · Homepage · AdGuard Home        │  │
│  │  Duplicati                                               │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
         ▲
         │ TwinGate (outbound only — no open router ports)
         │
┌────────┴────────┐
│  Remote clients  │
│  (TwinGate app)  │
└─────────────────┘
```

### Key Design Decisions

| Decision | Rationale |
|---|---|
| **Docker CE** as container runtime | All services run as containers — consistent deployment, easy updates |
| **No reverse proxy / no open ports** | Strictly local homelab; TwinGate handles remote access without exposing anything to the internet |
| **TwinGate Connector** | Zero-trust outbound-only tunnel; clients connect via TwinGate app to reach internal services |
| **AdGuard Home on port 53** | Internal DNS for `aria.local` hostnames; no conflict with router (different IPs). Disable `systemd-resolved` on Aria first. |
| **Ansible Vault** for secrets | All passwords and tokens stored encrypted; plaintext vault file is gitignored |
| **Pinned image versions** | All Docker images use specific semver tags — no `latest` drift |

---

## Directory Structure

```
ansible-setup-aria/
├── site.yml                         # Main playbook — entry point
├── inventory/
│   └── hosts.ini                    # Inventory: "aria" host (127.0.0.1, local connection)
├── group_vars/
│   └── all/
│       ├── vars.yml                 # Public variable mappings (safe to commit)
│       └── vault.yml                # Encrypted secrets — NEVER commit plaintext
└── roles/
    ├── docker/                      # Role: install & configure Docker CE
    │   ├── tasks/main.yml           # Remove old Docker → install → configure systemd
    │   ├── handlers/main.yml        # Restart / stop Docker handlers
    │   ├── templates/
    │   │   └── docker-override.conf.j2   # systemd override: always restart, 5s delay
    │   └── vars/main.yml            # Docker version, user, packages
    └── services/                    # Role: deploy all service stacks
        ├── meta/main.yml            # Declares dependency on docker role
        ├── tasks/
        │   ├── main.yml             # Loop over all stacks
        │   └── deploy_stack.yml     # Per-stack: create dir → configs → start → health
        ├── vars/main.yml            # All service image/version/port variables
        └── templates/               # One subdirectory per stack
            ├── monitoring/          # Prometheus, Grafana, Node Exporter, cAdvisor,
            │   │                    # Loki, Promtail, Alertmanager
            │   ├── docker-compose.yml.j2
            │   ├── prometheus.yml.j2
            │   ├── alerting_rules.yml.j2
            │   ├── alertmanager.yml.j2
            │   ├── loki-config.yml.j2
            │   └── promtail-config.yml.j2
            ├── portainer/
            │   └── docker-compose.yml.j2
            ├── twingate/
            │   └── docker-compose.yml.j2
            ├── ntfy/
            │   └── docker-compose.yml.j2
            ├── adguard/
            │   └── docker-compose.yml.j2
            ├── semaphore/
            │   └── docker-compose.yml.j2
            ├── vaultwarden/
            │   └── docker-compose.yml.j2
            ├── homepage/
            │   └── docker-compose.yml.j2
            ├── uptime-kuma/
            │   └── docker-compose.yml.j2
            ├── authelia/
            │   ├── docker-compose.yml.j2
            │   └── authelia-config.yml.j2
            ├── dockge/
            │   └── docker-compose.yml.j2
            ├── watchtower/
            │   └── docker-compose.yml.j2
            └── duplicati/
                └── docker-compose.yml.j2
```

---

## Deployed Services

### Phase 1 — Foundation

| Service | Stack | Port | Purpose |
|---|---|---|---|
| **TwinGate Connector** | `twingate` | *(none)* | Zero-trust remote access — outbound only, no router ports needed |
| **ntfy** | `ntfy` | 8088 | Self-hosted push notification server; receives alerts from Alertmanager, Uptime Kuma, Watchtower |
| **AdGuard Home** | `adguard` | 3001 (web), 53 (DNS) | Network DNS + ad blocking + internal `aria.local` hostname resolution |

### Phase 1 (cont.) — Monitoring Stack

All containers run in the same Docker Compose project (`monitoring`) and communicate by service name.

| Service | Port | Purpose |
|---|---|---|
| **Prometheus** | 9090 | Metrics collection and time-series storage; scrapes all exporters every 15s |
| **Grafana** | 3000 | Dashboards; data sources: Prometheus (metrics) + Loki (logs) |
| **Node Exporter** | *(internal)* | Host-level metrics: CPU, RAM, disk, network |
| **cAdvisor** | *(internal)* | Per-container resource metrics |
| **Loki** | 3100 | Log aggregation; 31-day retention enforced by compactor |
| **Promtail** | *(internal)* | Scrapes Docker container logs + `/var/log/syslog` + `/var/log/auth.log`, ships to Loki |
| **Alertmanager** | 9093 | Routes Prometheus alerts to ntfy webhooks |

**Alert rules defined** (`alerting_rules.yml`):
- `HostDown` — any scrape target unreachable for 1m (critical)
- `HighCpuUsage` — CPU > 85% for 5m (warning)
- `HighMemoryUsage` — RAM > 85% for 5m (warning)
- `DiskSpaceLow` — disk > 80% full (warning)
- `DiskSpaceCritical` — disk > 95% full (critical)
- `ContainerHighCpu` — container CPU > 80% for 5m (warning)
- `ContainerHighMemory` — container memory > 80% of limit (warning)

### Phase 2 — Operations

| Service | Stack | Port | Purpose |
|---|---|---|---|
| **Portainer CE** | `portainer` | 9443 | Docker container/image/volume management UI |
| **Semaphore** | `semaphore` | 3002 | Ansible web UI — run `site.yml`, manage inventory, view logs |
| **Vaultwarden** | `vaultwarden` | 8181 | Self-hosted Bitwarden-compatible password manager |
| **Uptime Kuma** | `uptime-kuma` | 3004 | HTTP/TCP/ping uptime monitoring for all internal services |
| **Homepage** | `homepage` | 3003 | Unified service dashboard with live Docker and Prometheus stats |

### Phase 3 — Polish

| Service | Stack | Port | Purpose |
|---|---|---|---|
| **Authelia** | `authelia` | 9091 | SSO / identity provider (OIDC); file-based users, SQLite storage |
| **Dockge** | `dockge` | 5001 | Docker Compose stack manager — create, edit, start, stop stacks via UI |
| **Watchtower** | `watchtower` | *(none)* | Checks for new image versions daily at 4am; sends update notifications to ntfy |
| **Duplicati** | `duplicati` | 8200 | Scheduled backup of Docker volumes and service configs |

---

## Deployment Order

The `services` role deploys stacks in this loop order, which respects dependencies:

```
1. twingate      ← remote access first (makes server reachable before anything else)
2. ntfy          ← notification backbone (alertmanager needs this)
3. adguard       ← DNS (internal hostnames available for all subsequent services)
4. monitoring    ← observability (prometheus, grafana, loki, etc.)
5. portainer     ← container management
6. semaphore     ← Ansible UI (can self-manage Aria from Aria)
7. vaultwarden   ← password manager
8. uptime-kuma   ← start monitoring services once they exist
9. homepage      ← dashboard (all services must exist to list them)
10. authelia     ← SSO
11. dockge       ← stack manager
12. watchtower   ← auto-updates (last, so it doesn't update before deploy finishes)
13. duplicati    ← backups (last, to back up everything already deployed)
```

---

## Prerequisites

### On the Ansible controller (your machine)

```bash
pip install ansible
ansible-galaxy collection install community.docker
```

### On Aria (the target server)

- Ubuntu 22.04 LTS or 24.04 LTS
- A user named `deploy` with passwordless sudo
- SSH key-based access from the controller, OR run locally on Aria itself
- **Port 53 conflict:** Ubuntu's `systemd-resolved` binds to `127.0.0.53:53`. Before deploying AdGuard, disable it:

```bash
sudo systemctl disable --now systemd-resolved
sudo rm /etc/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

After AdGuard is running, point `/etc/resolv.conf` to `127.0.0.1`.

---

## Secrets & Vault

All sensitive values live in `group_vars/all/vault.yml`, which is **gitignored in plaintext**. The file `group_vars/all/vars.yml` maps each secret to its `vault_*` counterpart and is safe to commit.

### Secrets managed by vault

| Variable | Description |
|---|---|
| `vault_grafana_admin_password` | Grafana admin login |
| `vault_semaphore_admin_password` | Semaphore admin login |
| `vault_semaphore_encryption_key` | Semaphore access key encryption (independent 32-char key) |
| `vault_twingate_network` | TwinGate network FQDN (e.g. `your-org.twingate.com`) |
| `vault_twingate_access_token` | TwinGate connector access token |
| `vault_twingate_refresh_token` | TwinGate connector refresh token |
| `vault_authelia_jwt_secret` | Authelia JWT signing secret (64+ chars) |
| `vault_authelia_session_secret` | Authelia session encryption secret |
| `vault_authelia_storage_encryption_key` | Authelia SQLite encryption key |

### Vault workflow

```bash
# 1. Fill in plaintext secrets
nano ansible-setup-aria/group_vars/all/vault.yml

# 2. Encrypt the file
ansible-vault encrypt ansible-setup-aria/group_vars/all/vault.yml

# 3. Edit secrets later
ansible-vault edit ansible-setup-aria/group_vars/all/vault.yml

# Optional: store vault password in a file (gitignored)
echo "your-vault-password" > .vault_pass
chmod 600 .vault_pass
```

Generate random secrets with:
```bash
openssl rand -hex 32   # for 64-char hex strings (Authelia secrets)
openssl rand -hex 16   # for 32-char hex strings (Semaphore key)
```

---

## Running the Playbook

```bash
cd ansible-setup-aria

# With vault password prompt
ansible-playbook site.yml -i inventory/hosts.ini --ask-vault-pass

# With vault password file
ansible-playbook site.yml -i inventory/hosts.ini --vault-password-file ../.vault_pass

# Dry run (check mode)
ansible-playbook site.yml -i inventory/hosts.ini --ask-vault-pass --check

# Only run a specific stack (using tags — add tags to site.yml to enable)
ansible-playbook site.yml -i inventory/hosts.ini --ask-vault-pass --extra-vars "stack_name=monitoring"
```

### What the playbook does

1. **`docker` role** — Removes old Docker packages, installs Docker CE from the official repository, adds the `deploy` user to the docker group, deploys a systemd override for automatic restart on failure.
2. **`services` role** — For each stack in order:
   - Creates `/opt/services/{stack_name}/`
   - Deploys `docker-compose.yml` from the role's `templates/{stack_name}/` directory
   - Deploys all extra config files (e.g. `prometheus.yml`, `loki-config.yml`) **before** starting containers
   - Starts the stack with `docker compose up`
   - Restarts the stack if any file changed
   - Health-checks with `docker compose ps` and auto-recovers if unhealthy

---

## Port Reference

| Port | Protocol | Service | Stack |
|---|---|---|---|
| 53 | TCP/UDP | AdGuard Home DNS | `adguard` |
| 3000 | TCP | Grafana | `monitoring` |
| 3001 | TCP | AdGuard Home web UI | `adguard` |
| 3002 | TCP | Semaphore | `semaphore` |
| 3003 | TCP | Homepage | `homepage` |
| 3004 | TCP | Uptime Kuma | `uptime-kuma` |
| 3100 | TCP | Loki | `monitoring` |
| 5001 | TCP | Dockge | `dockge` |
| 8088 | TCP | ntfy | `ntfy` |
| 8181 | TCP | Vaultwarden | `vaultwarden` |
| 8200 | TCP | Duplicati | `duplicati` |
| 9090 | TCP | Prometheus | `monitoring` |
| 9091 | TCP | Authelia | `authelia` |
| 9093 | TCP | Alertmanager | `monitoring` |
| 9443 | TCP | Portainer | `portainer` |

---

## Adding a New Service

1. **Create the template directory:**
   ```
   roles/services/templates/{stack-name}/docker-compose.yml.j2
   ```

2. **Add any extra config files** (mounted as volumes) in the same directory:
   ```
   roles/services/templates/{stack-name}/my-config.yml.j2
   ```
   These are automatically deployed before the container starts.

3. **Add image/version/port variables** to `roles/services/vars/main.yml`.

4. **Add secrets** (if any) to `group_vars/all/vault.yml` and map them in `group_vars/all/vars.yml`.

5. **Add the stack name** to the loop in `roles/services/tasks/main.yml` at the appropriate phase.

6. Re-run the playbook — the new stack will be deployed automatically.