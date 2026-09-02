---
> **🌐 Read this in [Italiano](README.it.md)**

---


# How my ecosystem works: one vault, two servers, one agent and four backup copies

My personal ecosystem is a full homelab: an Obsidian wiki that acts as
memory, a Hetzner server as data center, a Raspberry Pi as home automation
hub, and an AI agent (Hermes) that lives inside the system and administers it.

Two physical servers, about twenty containers, a photo library on a dedicated
storage and backups replicated in four locations. All without any port exposed
to the Internet: services listen on internal networks and the only entrance is
a VPN.

This article describes how it works — the network, the containers, the mounts,
the data flow, the backup and the role of the agent that coordinates everything.

![Technical diagram of the ecosystem: DROP firewall with Tailscale as the only entrance, Hermes container with RW/RO mounts, Immich photo flow to Storagebox and backups in 4 locations](files/ecosistema-360-architecture-diagram.png)

> *Caption: how the ecosystem works at a glance — at the top the security perimeter (Internet blocked, DROP firewall, Tailscale VPN as the only entrance); in the center the Hetzner server with Hermes and its RW/RO mounts, the 15 containers and the Immich flow to Storagebox; on the right the mini-server with the home services and the Pi storage (SD + external HDD); at the bottom the nightly backup chain with the 4 locations (local Hetzner repo → Storagebox, SD, HDD). Each color indicates a category: knowledge, cloud/storage, home/on-prem backup, API/VPN, agent/security, sync.*

---

## 1. The big picture

The ecosystem is made of three intertwined layers:

```
┌────────────────────────────────────────────────────────────────────┐
│                        ECOSYSTEM                                  │
│                                                                    │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐    │
│   │  Vault       │◄──►│  Hetzner     │◄──►│  Mini-server     │    │
│   │  (wiki)      │    │  (server)    │    │  (Raspberry Pi)  │    │
│   │  /obsidian-  │    │  /cloud-     │    │  /mini-server/   │    │
│   │  vault/      │    │  stack/      │    │  (smart home)    │    │
│   └──────┬───────┘    └──────┬───────┘    └────────┬─────────┘    │
│          │                   │                    │              │
│          └───────────────────┼────────────────────┘              │
│                              │                                    │
│                    ┌─────────▼─────────┐                         │
│                    │      Hermes       │                         │
│                    │   (AI agent)      │                         │
│                    │  workspace + tool │                         │
│                    └───────────────────┘                         │
└────────────────────────────────────────────────────────────────────┘
```

| Layer | Role | Where it lives |
|---------|-------|-----------|
| **Vault** | Persistent memory: wiki, manuals, procedures, career | Hetzner Storagebox, synced on 4 devices |
| **Hetzner server** | Personal data center: containers, data, automations | VPS (CX22, 4 cores, 7.6 GB RAM + 4 GB swap) |
| **Mini-server** | Home: smart home, DNS, monitoring, backup | Raspberry Pi 4 (ARM64, 3.7 GB RAM, zram) |
| **Hermes** | AI agent living in the cloud, coordinates and documents | Container on Hetzner |

The architectural principle is **data/service separation**: knowledge and
heavy data (photos, videos) live on dedicated storage, not on the servers'
disks. This way a hardware failure doesn't take the data down with the service.

---

## 2. The network: how two distant servers work as one

### 2.1 Zero public ports: the firewall

All services listen on local ports or on the internal Docker network.
The Hetzner server firewall is in **DROP on INPUT** mode: everything is closed
by default, and only the `tailscale0` and `docker0` interfaces are allowed.

| Firewall rule | Effect |
|-----------------|---------|
| DROP policy on INPUT | No port open by default |
| Only `tailscale0` allowed | Only entrance: the Tailscale VPN |
| Only `docker0` allowed | Internal Docker networks work |
| No port on public IP | Zero external attack surface |
| SSH only via Tailscale | Encrypted remote access, never on the Internet |

Remote access always happens **through the VPN (WireGuard)**: containers talk
to each other over the `casa_net` Docker network (by container name), the two
servers talk over Tailscale.

### 2.2 The networks

| Network | Type | Use |
|------|------|-----|
| `casa_net` | Internal Docker network (Hetzner) | Containers talk by name (Hermes ↔ Honcho ↔ NPM ↔ Immich...) |
| **Tailscale** | WireGuard VPN | Secure remote access to both servers |
| **SSHFS** | Remote mount | Hetzner mounts the Pi directory read-only (`/mini-server/`) |
| **Syncthing** | Peer-to-peer | Syncs the vault on 4 devices (Hetzner, laptop, 2 iPhones) |

The mini-server **does not use** Syncthing: it is reachable from Hetzner via
SSHFS and Tailscale. Its services (AdGuard, Fenrus, Uptime-Kuma, Home
Assistant) listen on the `host` network or the default bridge and are
reachable via Tailscale.

---

## 3. The containers: what runs where

### 3.1 Hetzner server — 15 live containers

| Container | Role |
|-----------|-------|
| **hermes** | The main agent (workspace + API gateway :8642) |
| **hermes-webui** | Agent web interface (:8787) |
| **twin instance (Elisa)** | Separate, isolated agent (no docker socket, no system logs) |
| **nginx-proxy-manager** | Reverse proxy: routes exposed services on 127.0.0.1 |
| **syncthing** | Vault synchronization (host network) |
| **ollama-embedding** | Local embeddings for AI memory (nomic-embed-text) |
| **honcho** (4 containers) | AI memory: API + deriver + PostgreSQL pgvector + Redis |
| **immich** (4 containers) | Photo/video library: server + ML + Postgres + valkey |
| **backrest** | restic backup management UI (:9898) |
| **arcane** | Container management panel (:3552), central hub |
| **actual-budget** | Finance management (127.0.0.1:5006 only) |

Notable details:
- **NPM** is the only container with ports exposed on all interfaces (80/443),
  but it routes to services on 127.0.0.1: this is not an exposure, it's the
  internal proxy.
- **The twin instance (separate profile)** is an isolated agent with a reduced
  perimeter: it has no docker socket, cannot read `/var/log/`, and has no
  access to `/cloud-stack/`, `/mini-server/` or the Storagebox data. Only its
  own vault and home.
- **syncthing** is on the only host-network container: it doesn't need `casa_net`.

### 3.2 Mini-server — 7 services (active compose)

| Service | Role | Network |
|----------|-------|------|
| **Home Assistant** (+ Mosquitto + Zigbee2MQTT) | Home automation: lights, alarm, HomePod, sensors | host / default |
| **AdGuard Home** | DNS server, blocks ads and trackers on the LAN | host |
| **Fenrus** | Custom dashboard (:3002) | default bridge |
| **Uptime-Kuma** | Uptime monitoring of both servers (:3001) | default bridge |
| **Arcane-agent** | Bridge between Pi containers and the Arcane hub on Hetzner (:3553) | default bridge |
| **hermes-ai** | Business twin agent, isolated (:8642, on casa_net) | casa_net |

The business twin on the Pi only has access to `/business/` and its own home:
no docker socket, no system logs, no vault. Deliberate isolation to separate
the personal domain from the business one.

---

## 4. The Hermes container mounts: what the agent sees

The agent doesn't have an "all or nothing" access: it has a **mount map**
defined in docker-compose. Every area has a precise permission.

```
hermes (container)
├── /obsidian-vault          RW   ← Storagebox (vault wiki, CWD)
├── /opt/data                RW   ← home (config, skills, memories, sessions)
├── /opt/hermes              RW   ← source code (shared volume)
├── /var/run/docker.sock     RW   ← Hetzner container management
├── /mini-server             RO   ← Pi directory via SSHFS
├── /var/log                 RO   ← Hetzner system logs
├── /cloud-stack             RO   ← Hetzner infrastructure (compose, data)
└── /dati-container-storagebox  RO ← container data (Immich photos/videos) on Storagebox
```

| Mount | Access | What Hermes can do |
|-------|---------|-----------------------|
| `/obsidian-vault/` | RW | Read and write the vault (with Plan/Act-Mode: only on request) |
| `/opt/data/` | RW | Home: configuration, skills, persistent memory |
| `/var/run/docker.sock` | RW | Manage Hetzner containers (logs, start, stop, restart) |
| `/mini-server/` | RO | Read Pi config, logs and data — **not** modify |
| `/var/log/` | RO | Analyze system logs |
| `/cloud-stack/` | RO | Read compose files and Hetzner infrastructure data |
| `/dati-container-storagebox/` | RO | Read the photo/video library (heavy data) |

Important constraints:
- Hermes **does not have** the mini-server docker socket: it can read the Pi
  data, but cannot manage its containers (a human does that).
- It can **write** only in `/obsidian-vault/` and `/opt/data/`; everything else
  is read-only.
- The PostgreSQL databases (Honcho, Immich) are not directly accessible:
  they are queried **only via API** on the `casa_net` network containers.

### The interaction cycle

Hermes operates in two modes:
- **Plan-Mode** (default): reads, analyzes, proposes. Writes nothing.
- **Act-Mode** (only on explicit request): executes changes, with reporting.

To orient itself it uses a context hierarchy: `SOUL.md` (identity) →
`AGENTS.md` (operating rules) → `guide` (structure) → `README` (detail).
When it has to operate on a domain, it reads that domain's guide first.

---

## 5. The knowledge: the vault and the photo library

### 5.1 The Obsidian vault

The vault lives on **Storagebox** (`/obsidian-vault/vault/llm-wiki-amir/`) and
is synced by Syncthing on 3 devices: Hetzner, Amir's laptop, Amir's iPhone. It
contains:

| Area | Content | Privacy |
|------|-----------|---------|
| `mondo-amir/` | Manuals, career, publications | 🔴🟡 |
| `llm-wiki/` | Public wiki (163 pages: entities + concepts + sources) | 🟢 |
| `inbox/` | Reports, agenda, mail | 🔴🟡 |
| `.hermes/` | Scripts, skills, tmp | 🔴 |
| Guides | `vault-guide.md`, `hetzner-guide.md`, `mini-server-guide.md` | 🟢 |

The vault is separated from the servers: knowledge doesn't depend on the
hardware hosting it. That's the **data/service separation** principle.

### 5.2 Immich and the Storagebox: where heavy photos end up

The photo/video library (Immich) is the emblematic case of data separation:

```
Immich stack (4 containers on Hetzner)
│
├── Photos/videos (heavy data) ──► /mnt/storagebox/dati_container/immich_library/
│       (Storagebox 1 TB, mounted RO for Hermes as /dati-container-storagebox/)
│
└── Metadata/DB (small data) ──► /opt/cloud-stack/data/immich/postgres/
        (Hetzner local disk, NOT directly readable → API only)
```

`UPLOAD_LOCATION` points to `/mnt/storagebox/dati_container/immich_library`.
The library structure:

```
immich_library/
├── library/admin/<year>/<ISO-date>/   # Original photos/videos
├── upload/                             # Uploads waiting for processing
├── thumbs/                             # Thumbnails/previews
├── encoded-video/                      # Transcoded videos
├── profile/                            # Profile photos
└── backups/                            # Automatic DB backups
```

Photos live separately from the server disk for a simple reason: the library
is **heavy**, media content grows fast and must not fill an 80 GB disk. The
1 TB Storagebox is the storage designed for the bulk. The metadata (the
"light" archive) stays on the server's fast disk.

---

## 6. The backup: four locations, every night

### 6.1 The strategy

The backup system is centralized on the Hetzner server and uses **restic**
(incremental, deduplicated, encrypted snapshots) to create the backups, and
**rclone** to replicate them. Every night, in sequence:

| Time | Script | What it does |
|-----|--------|---------|
| 03:00 | `script-dati-foto.sh` | Backup of photos/container data (from Storagebox) → **Pi external HDD** |
| 04:45 | `script-hetzner.sh` | Backup of `/opt/cloud-stack` (clean Docker stop/start) |
| 04:55 | `script-miniserver.sh` | Backup of mini-server data via SSHFS |
| 05:10 | `script-sync-backup.sh` | rclone sync of repositories → **3 remote destinations** |
| Sun 02:00 | `script-restic-check.sh` | Integrity check of restic repositories |

### 6.2 The four locations

Every restic repository exists in **4 copies**:

| # | Location | Where |
|---|-----------|------|
| 1 | **Local repository** | `/opt/cloud-stack/script-backups/backups/` (Hetzner disk) |
| 2 | **Storagebox** | Cloud replica via rclone |
| 3 | **Mini-server SD** | `/mini-server/copia-backup/` |
| 4 | **Mini-server external HDD** | `/mini-server/hdd-esterno/copia-backup/` |

The repositories contained are two:
- `server-hetzner/` — snapshot of `/opt/cloud-stack`
- `mini-server/` — snapshot of Pi data (via SSHFS)

### 6.3 The role of the mini-server external HDD

The 500 GB external HDD (`/dev/sda1`) has two complementary functions:

| Folder | What it contains |
|----------|---------------|
| `backup-foto-dati/` | **Dedicated backup of photos/container data** (03:00 script, restic, retention 3+4+12+2y) |
| `copia-backup/` | Replicas of the restic repositories (server-hetzner + mini-server) |

It's the "offline" copy of the bulk: the photo library (the largest part)
is replicated on the HDD, which is not the Pi system disk and doesn't touch
the 59 GB SD. No heavy file occupies space on the SD or the Hetzner disk.

### 6.4 Retention and verification

| Repository | Retention |
|------------|-----------|
| server-hetzner / mini-server | 7 daily + 4 weekly + 6 monthly (--prune) |
| foto-dati | 3 daily + 4 weekly + 12 monthly + 2 yearly |

The scripts use lock files to avoid concurrent executions, an anti-hang
`timeout`, and `docker stop --timeout 600` to stop containers cleanly. The
mini-server script has a **pre-flight check on the SSHFS mount**: if the
mount doesn't respond, the backup is cancelled and the containers are not
stopped. Outcomes land in `journalctl` with dedicated tags
(`backup-hetzner`, `backup-sync-rclone`, `backup-dati-foto`, `restic-check`...).

---

## 7. The periodic operations: Hermes crons

The agent is not only on-demand: it has a system of periodic automations that
every night and every week keep the ecosystem alive.

### 7.1 Hermes crons (on Hetzner)

| Job | Time | Function |
|-----|--------|----------|
| **sync-cron-scripts** | 05:55 | Aligns cron scripts between vault and home |
| **daily-check** | 06:00 | 6 phases: manuals sync, publications sync, indexes, skills, server health |
| **daily-mail** | 06:30 | Daily email triage (via Composio) |
| **daily-briefing** | 07:00 | Morning summary (email, logs, agenda) |
| **weekly-jobs** | Mon 03:00 | LinkedIn job offers monitoring |
| **Wiki lint** | Sun 11:00 | Checks and refines the wiki (broken links, orphans, tags) |
| **curriculum-knowledge-update** | Biweekly | Manuals analysis + market → skills report |
| **Documentation audit** | 1st odd month (paused) | Compares guides/READMEs vs reality (docker, mounts, cron) |

In CRON context, Hermes operates automatically and silently: it delivers
reports on Telegram/notification channels, without interaction. It doesn't use
Act-Mode in cron: changes always require human approval.

### 7.2 System crons (nightly backups)

In addition to the agent jobs, the system crons run the backups described in
§6.1 (03:00, 04:45, 04:55, 05:10, Sunday 02:00) and the daily health checks
that produce reports with structured sections (general information,
resources, temperatures, SMART, kernel/OOM, SSH, logs, Docker, backups,
updates) and a machine-readable JSON block.

---

## 8. The security: Hermes perimeter

The ecosystem security model boils down to a few principles:

| Principle | Implementation |
|-----------|-----------------|
| **Zero public ports** | DROP firewall on INPUT; only Tailscale + Docker |
| **Remote access VPN only** | SSH and services only via Tailscale |
| **Granular permissions** | The agent writes only in `/obsidian-vault/` and `/opt/data/`; the rest is RO |
| **Sensitive data out of public view** | 🔴 private zones (assets, profile, .env) never published |
| **Placeholders in public documents** | IP `<...>`, credentials `***` |
| **Frozen scripts** | Automation scripts are not modified directly |

Hermes perimeter is designed to minimize damage: it can read (and therefore
analyze) everything, but writes only where it must. The agent never decides on
its own to break something: Plan-Mode to analyze, Act-Mode only after explicit
approval.

---

## 9. Lessons and next steps

### Lessons

1. **Separate data from service**: photos on Storagebox, config on the server,
   backups on the HDD. A failure on one node doesn't touch the data.
2. **The agent with granular permissions**: read everywhere, write only where
   needed. That's the reason it can administer a real infrastructure.
3. **Four copies, weekly verification**: a backup only works if you know it
   works, and the Sunday `restic check` guarantees it.
4. **Documentation is part of the infrastructure**: guides and READMEs are
   consulted by the agent before every operation, and a periodic audit checks
   their alignment with reality.
5. **Skills as operational capital**: the agent's skills are modular,
   versioned and validated, managed by a dedicated skill that checks them
   against best practices.

### Next steps

- Re-activate the documentation audit in non-overlapping mode
- Publish the ecosystem guides as a series on GitHub/LinkedIn
- Extend monitoring (Pi temperatures) to the briefing KPIs
- Explore automation of decision processes (delegate roadmap)

---

## Residual risks

- **Single point of failure on the cloud**: if the Hetzner VPS dies, the
  "intellectual" ecosystem stops (the home keeps living on the Pi). Mitigation:
  4 backup copies, but recovery takes time.
- **Mini-server read-only for the agent**: if a fix is needed on the Pi, a
  human has to apply it. Acceptable latency, but structured.
- **Privacy**: the public wiki is managed by a free API that retains data for
  training → personal data lives elsewhere (🔴 zones).
- **Dependence on external providers**: cloud models, APIs, storage. The
  system is decentralized but not independent.

## Resources / Links

- [Repository (coming soon)]
- Related articles: wiki plugin (published), docker-compose best practices (in progress)

*Descriptive article based on the ecosystem's living documentation:
vault-guide.md, hetzner-guide.md, mini-server-guide.md, General Infrastructure
Manual, real docker-compose files and live crons.*