# Homelab

Last updated: 2026-10-06

IP addresses are replaced with placeholders such as `<nas-ip>`.

## Overview

The homelab is five servers running 67 Docker containers, mostly a self-hosted media stack. The Unraid NAS holds 99 TB of storage and runs 39 of the containers; four Ubuntu servers handle playback, transcoding, AI and testing. A Prometheus and Grafana stack on the Testing server monitors all of them (see [Monitoring](#monitoring)). One external Linode VPS, a VPN gateway, is monitored too but is not counted as one of the five.

| Host | IP | Role | OS | CPU | RAM | GPU |
| --- | --- | --- | --- | --- | --- | --- |
| ImranNas | `<nas-ip>` | Storage, downloads, \*arr apps, most services | Unraid OS 7.3.2 | Xeon E5-2687W v3, 40 threads | 62 GB | GTX 1650 Super |
| MYPHY-UBUNTU-MASTER-SERVER | `<master-ip>` | Transcoding (Tdarr node), tools | Ubuntu 24.04.5 | Xeon X5675, 24 threads | 125 GB | Tesla P4 |
| MYPHY-UBUNTU-MEDIA-SERVER | `<media-ip>` | Playback: Jellyfin, Plex, Komga, Navidrome | Ubuntu 24.04.5 (FIPS kernel) | Ryzen 5 5600G, 12 threads | 30 GB | RTX 3070 |
| MYPHY-UBUNTU-AI-SERVER | `<ai-ip>` | AI: subtitle-ai | Ubuntu 26.04.1 | Ryzen 5 5500, 12 threads | 14 GB | RTX 3070 (8 GB) |
| MYPHY-UBUNTU-TESTING-SERVER | `<testing-ip>` | Testing and monitoring: Dockhand, OmniRoute, Prometheus, Grafana | Ubuntu 24.04.5 | AMD RX-427BB, 4 threads | 6.7 GB | Radeon R7 (integrated) |

**External host:** SGVMP-UBUNTU-GATEWAY-SERVER is a Linode VPS (Ubuntu 24.04.5, 25 GB disk) at `<vps-ip>`. It is a VPN gateway running OpenVPN, WireGuard and Tailscale, plus Uptime Kuma and ntopng. It is monitored over its public IP and is outside the five-server count.

## Network and access

Three subnets are in use. `<storage-net>` is the storage network that carries NFS from the NAS to the Ubuntu servers.

| Subnet | Purpose | Hosts |
| --- | --- | --- |
| `<server-lan>` | Server LAN | Testing, Media, AI, Master |
| `<nas-lan>` | NAS LAN | ImranNas |
| `<storage-net>` | Storage network (NFS) | Media (10 GbE bond), AI, Master, NAS (bond) |

**NFS shares from the NAS**

| Client | Share | Mount point | Via |
| --- | --- | --- | --- |
| Master | /mnt/user/main_directory | /data | `<nas-storage-ip>` |
| Master | /mnt/user/private_directory | /mnt/private | `<nas-storage-ip>` |
| AI | /mnt/user/main_directory | /data | `<nas-storage-ip>` |
| Media | /mnt/user/main_directory | /data | `<nas-ip>` |

**Remote access:** the NAS runs Tailscale, a Twingate connector and a Cloudflare tunnel (cloudflared).

**SSH:** all five hosts accept an ECDSA P-384 key. The Media server runs a FIPS build of OpenSSH, which refuses ED25519 keys; ECDSA on NIST curves and RSA of 2048 bits or more are FIPS-approved.

## Server details

The NAS array is 67% full (66 TB of 99 TB). Every local disk on the other servers is under half full. Figures as of 2026-09-25.

### ImranNas

Unraid 7.3.2, kernel 6.18.38, Docker 29.5.3. Also on the storage network at `<nas-storage-ip>`.

| Storage | Size | Used |
| --- | --- | --- |
| Array: 8 data disks (disk1–disk8) | 8 × 11 TB | 64–76% each |
| User shares (/mnt/user) | 99 TB | 66 TB (67%) |
| Cache pool, NVMe (/mnt/cache) | 477 GB | 79 GB (17%) |
| Docker disk (/mnt/docker) | 932 GB | 37 GB (4%) |

### MYPHY-UBUNTU-MASTER-SERVER

Ubuntu 24.04.5, kernel 6.8.0-142, Docker 29.8.1. Also on the storage network at `<master-storage-ip>`. The hardware is a Dell PowerEdge R610, which has an iDRAC6 management controller; the NAS runs an iDRAC6 console container.

| Storage | Size | Used |
| --- | --- | --- |
| / (LVM) | 196 GB | 81 GB (44%) |
| /opt | 916 GB | 102 GB (12%) |
| /data and /mnt/private | NFS from NAS | see NAS |

### MYPHY-UBUNTU-MEDIA-SERVER

Ubuntu 24.04.5 with the FIPS kernel 6.8.0-142-fips, Docker 29.8.1. Also on the storage network at `<media-storage-ip>`, over a 10 GbE bond.

| Storage | Size | Used |
| --- | --- | --- |
| / (LVM) | 393 GB | 74 GB (20%) |
| /opt (bcache) | 3.6 TB | 223 GB (7%) |
| /srv | 5.5 TB | 108 GB (3%) |
| /data | NFS from NAS | see NAS |

### MYPHY-UBUNTU-TESTING-SERVER

Ubuntu 24.04.5, kernel 6.8.0-139, Docker 29.8.1.

| Storage | Size | Used |
| --- | --- | --- |
| / (LVM) | 177 GB | 23 GB (14%) |

### MYPHY-UBUNTU-AI-SERVER

Ubuntu 26.04.1, kernel 7.0.0-34, Docker 29.8.1, NVIDIA driver 595.91.07. Gigabyte B550M DS3H AC R2 board. Added 2026-09-27; also on the storage network at `<ai-storage-ip>`. Its Wi-Fi is also connected to the server LAN, alongside the wired connection.

| Storage | Size | Used |
| --- | --- | --- |
| / (LVM, NVMe) | 885 GB | 40 GB (5%) |
| /data | NFS from NAS | see NAS |

## Services

All 67 containers were running on 2026-10-06. Port is the host port for the web UI or API; a blank means none is published. Three services run on more than one host: metube, tdarr_node and docker-socket-proxy. The monitoring exporters (see [Monitoring](#monitoring)) also run on several hosts.

### Media playback and libraries

| Service | Host | Port | Image |
| --- | --- | --- | --- |
| Jellyfin | Media | host network | jellyfin/jellyfin |
| Plex | Media | host network | lscr.io/linuxserver/plex |
| Komga (comics) | Media | 25600 | gotson/komga |
| Komf (Komga metadata) | Media | 8085 | sndxr/komf |
| Suwayomi (manga) | NAS | 4567 | suwayomi-server |
| Immich (photos) | NAS | 8089 | imagegenius/immich |
| Immich Postgres | NAS | | immich-app/postgres 16 |
| Redis (Immich) | NAS | | redis:8-alpine |
| Navidrome (music) | Media | 4533 | deluan/navidrome |

### Requests, stats and sync

| Service | Host | Port | Image |
| --- | --- | --- | --- |
| Seerr (requests) | NAS | 5055 | seerr-team/seerr |
| Wizarr (invites) | NAS | 5690 | wizarrrr/wizarr |
| Tautulli (Plex stats) | NAS | 8181 | linuxserver/tautulli |
| Jellystat (Jellyfin stats) | NAS | 3001 | cyfershepard/jellystat |
| Jellystat DB | NAS | | postgres:15-alpine |
| Tracearr | NAS | 3000 | connorgallopo/tracearr |
| JellyPlex-Watched | NAS | | luigi311/jellyplex-watched |
| PlexTraktSync | NAS | | taxel/plextraktsync |

### Library automation (\*arr)

| Service | Host | Port | Image |
| --- | --- | --- | --- |
| Sonarr (TV) | NAS | 8989 | linuxserver/sonarr |
| Radarr (movies) | NAS | 7878 | linuxserver/radarr |
| Lidarr (music) | NAS | 8686 | linuxserver/lidarr |
| Bazarr (subtitles) | NAS | 6767 | linuxserver/bazarr |
| Prowlarr (indexers) | NAS | 9696 | linuxserver/prowlarr |
| FlareSolverr | NAS | 8191 | flaresolverr |

### Downloads

qBittorrent and NZBGet run inside Gluetun's network namespace, so their traffic goes through the VPN and their UIs are on Gluetun's ports 8080 and 6789.

| Service | Host | Port | Image |
| --- | --- | --- | --- |
| Gluetun (VPN) | NAS | 8080, 6789, 38554 | qmcgaw/gluetun |
| qBittorrent | NAS | via Gluetun | linuxserver/qbittorrent |
| NZBGet | NAS | via Gluetun | linuxserver/nzbget |
| MeTube | NAS | 8081 | alexta69/metube |
| MeTube | Media | 8081 | alexta69/metube |
| Downtify | NAS | 8000 | henriquesebastiao/downtify |

### Transcoding and subtitles

| Service | Host | Port | Image |
| --- | --- | --- | --- |
| Tdarr server | NAS | 8264, 8265 | haveagitgat/tdarr |
| Tdarr node | NAS | | haveagitgat/tdarr_node |
| Tdarr node | Master | | haveagitgat/tdarr_node |
| Tdarr Inform | NAS | 5004 | deathbybandaid/tdarr_inform |
| subtitle-ai | AI | 8099 | subtitle-ai:dev (local build) |
| Jellyfin SubSync | NAS | 8420 | marnalas/jellyfin-subsync-sidecar |

### Files and tools

| Service | Host | Port | Image |
| --- | --- | --- | --- |
| FileBot | NAS | 7813 | jlesage/filebot |
| Krusader | NAS | 5800 | jlesage/krusader |
| Firefox | NAS | 3010, 3011 | linuxserver/firefox |
| media-duplicate-finder | Master | 8090 | local build |
| iDRAC6 console | NAS | 5801 | domistyle/idrac6 |
| Vaultwarden | NAS | | vaultwarden/server |

### Infrastructure and testing

| Service | Host | Port | Image |
| --- | --- | --- | --- |
| Cloudflared (tunnel) | NAS | | cloudflare/cloudflared |
| Twingate connector | NAS | | twingate/connector |
| docker-socket-proxy | NAS | 2375 | tecnativa/docker-socket-proxy |
| docker-socket-proxy | Master | 2375 | tecnativa/docker-socket-proxy |
| docker-socket-proxy | AI | 2375 | tecnativa/docker-socket-proxy |
| Dockhand | Testing | 3000 | fnsys/dockhand |
| Dockhand Postgres | Testing | | postgres:16-alpine |
| OmniRoute | Testing | 20128 | diegosouzapw/omniroute |

### Monitoring

| Service | Host | Port | Image |
| --- | --- | --- | --- |
| Prometheus | Testing | 9090 | prom/prometheus |
| Grafana | Testing | 3030 | grafana/grafana |
| cAdvisor | Testing, Media, Master, AI, NAS | 18080 | ghcr.io/google/cadvisor |
| node_exporter (container) | Master, AI, NAS | 9100 | prom/node-exporter |
| NVIDIA GPU exporter | Media, Master, AI, NAS | 9835 | utkuozdemir/nvidia_gpu_exporter |
| smartctl exporter | NAS | 9633 | prometheuscommunity/smartctl-exporter |

## Monitoring

Prometheus scrapes every host every 15 seconds and keeps 90 days of data. Grafana reads from it at `http://<testing-ip>:3030` (user `admin`; the password is in the `.env` file next to the compose file on Testing). Prometheus is at `http://<testing-ip>:9090`.

**What is scraped**

| Job | Targets | Notes |
| --- | --- | --- |
| node | all five servers and the VPS | Testing and Media run node_exporter as a systemd service; Master, AI, NAS and the VPS run it as a container |
| cadvisor | all five servers and the VPS | Per-container CPU, memory, network |
| nvidia_gpu | Media, Master, AI, NAS | GPU use, temperature, VRAM |
| smartctl | NAS | Disk SMART health and temperature, every 60 seconds |

Every target carries a `host` label (testing, media, master, ai, nas, vps).

**Grafana dashboards** (folder "Homelab", provisioned from JSON files): Node Exporter Full, cAdvisor, NVIDIA GPU Metrics, and smartctl. They came from grafana.com (IDs 1860, 14282, 14574, 20204). The smartctl dashboard is filtered to the NAS exporter.

**Alert rules** (`alerts.yml`, shown in Grafana under Alerting)

| Alert | Fires when | Severity |
| --- | --- | --- |
| HostDown | a host's node_exporter is unreachable for 2 minutes | critical |
| ExporterDown | any other exporter is unreachable for 5 minutes | warning |
| DiskSpaceLow | a filesystem has under 10% free for 10 minutes | warning |
| DiskSpaceCritical | a filesystem has under 5% free for 5 minutes | critical |

The disk rules skip tmpfs, overlay and similar pseudo-filesystems, and NFS mounts, so the NAS is not counted again on the hosts that mount it. No notification channel is configured yet, so alerts only appear in the Grafana and Prometheus UIs.

**Where the files are**

| Host | Compose file |
| --- | --- |
| Testing | `/opt/docker/compose/monitoring/compose.yaml` (also `prometheus/prometheus.yml`, `prometheus/alerts.yml`, `grafana/provisioning/`) |
| Testing data | `/opt/docker/appdata/monitoring/` (Prometheus, Grafana, dashboards) |
| Media, Master, AI | `/opt/docker/compose/exporters/compose.yml` |
| NAS | `/mnt/user/docker-compose/compose/monitoring/compose.yaml` |
| VPS | `/opt/docker/compose/exporters/compose.yml` |

To change scrape targets or alerts, edit the files on Testing and restart Prometheus: `cd /opt/docker/compose/monitoring && docker compose restart prometheus`.

**VPS access:** the VPS is scraped over its public IP. Ports 9100 and 18080 there accept only the home public IP, plus loopback and the `tailscale0`, `tun0` and `wg0` interfaces; everything else on `eth0` is dropped (IPv4 and IPv6). This is a separate `EXPORTER-IN` iptables chain, set at boot by `exporter-firewall.service` from `/usr/local/sbin/exporter-firewall.sh`. The home IP is PPPoE and can change. If it does, the VPS scrape fails and HostDown fires; update `ALLOW=` in that script and run `systemctl restart exporter-firewall` on the VPS. Testing cannot reach the VPS over Tailscale, because pfSense does not NAT LAN traffic into the tailnet.

**Older stack:** a previous Prometheus, Loki, Alloy, Grafana and Authentik stack (data to March 2026) is still on disk on Testing in `/opt/docker` (`compose.recovered`, `config/`, `data/`). It is not running and is not used by the current stack.

## Maintenance

The main follow-ups are the Media server's NFS route, pinning image versions, and adding a notification channel for alerts.

### Health check

This lists Docker containers on every host. It uses the SSH aliases from `~/.ssh/config`:

```bash
for h in myphy-testing myphy-media myphy-master myphy-ai myphy-nas; do
  echo "== $h"; ssh $h 'docker ps -a --format "{{.Names}}\t{{.Status}}"'
done
```

To check the monitoring itself, open Prometheus at `http://<testing-ip>:9090/targets`. All targets should be up.

The NAS is Unraid, so `systemctl` isn't available there; check Docker with `docker info` instead.

### Rotating the SSH key

1. Create a FIPS-compatible key: `ssh-keygen -t ecdsa -b 384 -f ~/.ssh/<name>`.
2. Append the `.pub` file to `~/.ssh/authorized_keys` on each host while the old key still works.
3. Test each host with `ssh -o IdentitiesOnly=yes -i ~/.ssh/<name> <host>`.
4. Add the new key to GitHub, then remove the old key's line from every `authorized_keys`.

### Known issues and follow-ups

- [ ] The Media server mounts `/data` from `<nas-ip>`, not over the 10 GbE storage network (`<nas-storage-ip>`) that Master uses.
- [x] Add a `~/.ssh/config` alias for the NAS, like the other four hosts have.
- [ ] Nearly every image uses the `latest` tag, so a pull can bring in breaking changes; pin versions for critical services (Vaultwarden, Immich).
- [ ] Watch NAS disk1 and disk2, the fullest array disks at 76% and 75%.
- [ ] Add a notification channel (Alertmanager, or Uptime Kuma on the VPS) so firing alerts send a message instead of only showing in Grafana.
- [ ] Set up a pfSense SNMP dashboard; the old stack had one, the current stack does not.
- [ ] The VPS exporter allow-list depends on the home public IP, which can change.
- [ ] The VPS allows SSH root login with a password; consider key-only login.
