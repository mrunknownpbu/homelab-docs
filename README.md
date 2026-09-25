# Homelab

Last updated: 2026-09-25

IP addresses are replaced with placeholders such as `<nas-ip>`.

## Overview

The homelab is four servers running 47 Docker containers, mostly a self-hosted media stack. The Unraid NAS holds 99 TB of storage and runs 34 of the containers; three Ubuntu servers handle playback, transcoding and testing.

| Host | IP | Role | OS | CPU | RAM | GPU |
| --- | --- | --- | --- | --- | --- | --- |
| ImranNas | `<nas-ip>` | Storage, downloads, \*arr apps, most services | Unraid OS 7.3.2 | Xeon E5-2687W v3, 40 threads | 62 GB | GTX 1650 Super |
| MYPHY-UBUNTU-MASTER-SERVER | `<master-ip>` | Transcoding (Tdarr node), subtitle AI, tools | Ubuntu 24.04.5 | Xeon X5675, 24 threads | 125 GB | Tesla P4 |
| MYPHY-UBUNTU-MEDIA-SERVER | `<media-ip>` | Playback: Jellyfin, Plex, Komga | Ubuntu 24.04.5 (FIPS kernel) | Ryzen 5 5600G, 12 threads | 30 GB | RTX 3070 |
| MYPHY-UBUNTU-TESTING-SERVER | `<testing-ip>` | Testing: Dockhand, OmniRoute | Ubuntu 24.04.5 | AMD RX-427BB, 4 threads | 6.7 GB | Radeon R7 (integrated) |

## Network and access

Three subnets are in use. `<storage-net>` is the storage network that carries NFS from the NAS to the Ubuntu servers.

| Subnet | Purpose | Hosts |
| --- | --- | --- |
| `<server-lan>` | Server LAN | Testing, Media, Master |
| `<nas-lan>` | NAS LAN | ImranNas |
| `<storage-net>` | Storage network (NFS) | Media (10 GbE bond), Master, NAS (bond) |

**NFS shares from the NAS**

| Client | Share | Mount point | Via |
| --- | --- | --- | --- |
| Master | /mnt/user/main_directory | /data | `<nas-storage-ip>` |
| Master | /mnt/user/private_directory | /mnt/private | `<nas-storage-ip>` |
| Media | /mnt/user/main_directory | /data | `<nas-ip>` |

**Remote access:** the NAS runs Tailscale, a Twingate connector and a Cloudflare tunnel (cloudflared).

**SSH:** all four hosts accept an ECDSA P-384 key. The Media server runs a FIPS build of OpenSSH, which refuses ED25519 keys; ECDSA on NIST curves and RSA of 2048 bits or more are FIPS-approved.

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

## Services

All 47 containers were running on 2026-09-25. Port is the host port for the web UI or API; a blank means none is published. Three services run on two hosts: metube, tdarr_node and docker-socket-proxy.

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

### Transcoding and subtitles

| Service | Host | Port | Image |
| --- | --- | --- | --- |
| Tdarr server | NAS | 8264, 8265 | haveagitgat/tdarr |
| Tdarr node | NAS | | haveagitgat/tdarr_node |
| Tdarr node | Master | | haveagitgat/tdarr_node |
| Tdarr Inform | NAS | 5004 | deathbybandaid/tdarr_inform |
| subtitle-ai | Master | 8099 | subtitle-ai:dev (local build) |
| subtitle-ai-translate-server | Media | 8091 | subtitle-ai:dev (local build) |
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
| Dockhand | Testing | 3000 | fnsys/dockhand |
| Dockhand Postgres | Testing | | postgres:16-alpine |
| OmniRoute | Testing | 20128 | diegosouzapw/omniroute |

## Maintenance

The main follow-ups are the Media server's NFS route and pinning image versions.

### Health check

This lists Docker containers on every host. It assumes SSH aliases named testing, media, master and nas in `~/.ssh/config`:

```bash
for h in testing media master nas; do
  echo "== $h"; ssh $h 'docker ps -a --format "{{.Names}}\t{{.Status}}"'
done
```

The NAS is Unraid, so `systemctl` isn't available there; check Docker with `docker info` instead.

### Rotating the SSH key

1. Create a FIPS-compatible key: `ssh-keygen -t ecdsa -b 384 -f ~/.ssh/<name>`.
2. Append the `.pub` file to `~/.ssh/authorized_keys` on each host while the old key still works.
3. Test each host with `ssh -o IdentitiesOnly=yes -i ~/.ssh/<name> <host>`.
4. Add the new key to GitHub, then remove the old key's line from every `authorized_keys`.

### Known issues and follow-ups

- [ ] The Media server mounts `/data` from `<nas-ip>`, not over the 10 GbE storage network (`<nas-storage-ip>`) that Master uses.
- [ ] Add a `~/.ssh/config` alias for the NAS, like the other three hosts have.
- [ ] Nearly every image uses the `latest` tag, so a pull can bring in breaking changes; pin versions for critical services (Vaultwarden, Immich).
- [ ] Watch NAS disk1 and disk2, the fullest array disks at 76% and 75%.
