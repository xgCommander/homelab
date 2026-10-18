# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A homelab Docker Compose configuration. Each service lives in its own subdirectory with a `compose.yaml`. Services are managed via Dockge at `/home/debian/homelab`.

## Common Commands

```bash
# Start/stop a service stack
docker compose -f <service>/compose.yaml up -d
docker compose -f <service>/compose.yaml down

# Bootstrap the shared network (run once on a new host)
bash xutils/bootstrap-networks.sh

# Format/normalize compose files (no args = every compose.yaml in the repo)
python3 xutils/fmt-compose.py [path/to/compose.yaml ...]
```

## Architecture

**Network:** `tk-proxy` (short for "traefik proxy") is the external bridge network that Traefik watches for container labels. **Any service that needs a hostname / public route must be on `tk-proxy`** — Traefik can only reach containers it shares a network with. Create it once per host with `xutils/bootstrap-networks.sh`. Exceptions to the default attachment:
- `beszel-agent` and `scanopy-daemon` run with `network_mode: host` (need to see host network for monitoring/scanning) — they don't need Traefik routing
- The `*arr` services and `flaresolverr` use `network_mode: container:qbittorrentvpn`, sharing the VPN container's namespace. Because `qbittorrentvpn` itself is attached to `tk-proxy`, Traefik reaches the *arr WebUIs through it — so their Traefik labels are declared on `qbittorrentvpn`, not on the individual services

**Docker socket:** Traefik, Homepage and Dozzle don't mount `/var/run/docker.sock`; they reach Docker through `socket-proxy` (`lscr.io/linuxserver/socket-proxy`, stack `socket-proxy/`) at `tcp://socket-proxy:2375` on the internal `socket-proxy` network. It is GET-only (`POST: 0`) with CONTAINERS, EVENTS, INFO, NETWORKS, ALLOW_LOGS enabled. The `socket-proxy` stack must be up before those three start. Dockge still mounts the socket read-write (it needs to run compose); beszel-agent and scanopy-daemon mount it read-only (host network).

**Private networks:** `scanopy-postgres` is only on the internal `scanopy-internal` network (scanopy-server joins both it and tk-proxy). Its password is `POSTGRES_PASSWORD` in `scanopy/.env`; changing it requires `ALTER USER` in the running DB, since the env var only applies at first init.

**Reverse proxy:** Traefik handles TLS termination and routing. Its API is not insecure: `api.insecure`/`api.debug` are `false`, and `api@internal` is served only through basic-auth routers — `traefik` (dashboard on `traefik.local.<domain>`) and `traefik-internal` (entrypoint `traefik` on `:8080`, used by Homepage's Traefik widget with the dashboard credentials). Each service declares its own routing rules and port via Docker labels. The domain is purchased and managed in Cloudflare (DNS only — no Cloudflare tunnel). Three subdomains point to different entry points:
- `*.local.<domain>` — points to `matrix`'s static LAN IP; only reachable on the local network
- `*.ts.<domain>` — points to `domainforge` (Raspberry Pi Zero 2W), static LAN IP, configured via Tailscale DNS settings
- `*.cf.<domain>` — points to `matrix`'s Tailscale IP; same machine as `.local.` but accessed over Tailscale

Traefik config is split between two files in `traefik/container_data/traefik/`:
- `traefik.yml` — static config (entrypoints, providers, certresolver definitions)
- `config.yml` — file-provider dynamic config containing routes for services hosted on `domainforge` (the Pi). Services running on `matrix` declare their own routes via Docker labels; services on the Pi can't, so they're defined here as static file routes pointing to the Pi's backends.

TLS certificates are obtained via Cloudflare DNS challenge (API token in `traefik/.env`). Cert state lives in `traefik/container_data/traefik/acme.json` — losing this file forces full reissuance, which is rate-limited by Let's Encrypt; treat it as critical.

**LAN:** Subnet is hardcoded in the router. The same value must stay in sync in `qbittorrent_arr/.env` (`LAN_NETWORK`) — it's the only subnet the VPN killswitch lets bypass the tunnel, so LAN clients can reach the qBit/`*arr` WebUIs.

**LAN DNS:** `domainforge` (Pi Zero 2W, DietPi) runs AdGuard Home. The router uses it as primary DNS; secondary is `1.1.1.1`. Internal hostname resolution and ad-blocking happen there.

**Tailscale ACL:** Two Tailscale accounts are in use:
- `tag:owner` (admin account) — full access to all ports and SSH on owner-tagged devices
- `tag:member` (shared account used by friends) — restricted to the server's port 80 and 443 only (Traefik ingress), cannot reach any other device or port

The effect is that friends can reach services exposed through Traefik on the `.cf.` domain but have no lateral movement within the tailnet.

**VPN-isolated stack (`qbittorrent_arr`):** All traffic for the *arr services and FlareSolverr is forced through the OpenVPN tunnel inside `qbittorrentvpn` (ProtonVPN). Killswitch is enforced by the VPN container's iptables rules — if the tunnel drops, none of the dependent containers can reach the internet. `VPN_INPUT_PORTS` in `qbittorrent_arr/compose.yaml` whitelists the WebUI ports through the firewall.

`qbittorrentvpn` has a healthcheck (tun0 up + WebUI answering), and every dependent service uses `depends_on: {qbittorrentvpn: {condition: service_healthy, restart: true}}`. Without this, recreating `qbittorrentvpn` leaves the dependents attached to a dead network namespace ("joining network namespace of container: No such container") and they must be force-recreated.

qBittorrent WebUI auth: the bypass whitelist is the LAN subnet only, and reverse-proxy support is on with `172.18.0.0/16` (tk-proxy) as trusted proxies, so qBit judges clients by Traefik's `X-Forwarded-For`. LAN browsers skip login; Tailscale (`.ts`, incl. `tag:member` friends) must log in. **Never add `172.18.0.0/16` to the auth whitelist** — every request through Traefik comes from that subnet, so it would disable auth for everyone. Homepage's qBittorrent widget authenticates with username/password. binhex rewrites `config/openvpn/credentials.conf` with mode 775 (from `UMASK`) on every start; OpenVPN warns about it.

**Jellyfin:** Uses Intel VA-API via `/dev/dri/renderD128` with `group_add: '105'` (the host `render` group) for H.264 hardware transcoding. JellySearch intercepts search queries at the Traefik router level and proxies them to MeiliSearch. Transcode temp is on the SSD (`./container_data/jellyfin/transcode`), cache on SSD (`./container_data/jellyfin/cache`).

The iGPU is Haswell-ULT (HD 4400, i7-4510U) — Gen7.5, driven by the legacy `i965` VA-API driver (`iHD` does not support it; its init failure in the ffmpeg logs is expected noise). Two hard limits follow, and both are easy to trip from the Jellyfin admin UI:
- **`EnableIntelLowPowerH264HwEncoder` must stay `false`.** Haswell exposes no `VAEntrypointEncSliceLP` — low-power/VDENC encoding starts at Skylake. With it on, every transcode dies instantly (`No usable encoding entrypoint found for profile VAProfileH264High` → `FFmpeg exited with code 218`), which surfaces to users as a playback error the moment anything forces a transcode (e.g. seeking).
- **No HW HEVC/VP9/AV1 decode and no 10-bit.** vainfo exposes decode for MPEG-2, H.264 and VC-1 only (no MPEG-4 Part 2, no VP8), so `HardwareDecodingCodecs` must list only h264/mpeg2video/vc1. HEVC transcodes therefore software-decode; a single 1080p HEVC 10-bit stream runs ~2.1x realtime, so two concurrent ones will stutter. Full-HW H.264 transcodes run ~11x.

The machine also has an NVIDIA GeForce 840M (GM108, driver 535, `nvidia` container runtime installed). It is useless for Jellyfin: GM108 has no NVENC (`h264_nvenc` fails with -22), CUDA hwaccel init fails in jellyfin-ffmpeg, and Maxwell gen1 NVDEC has no HEVC anyway. Don't wire it into the Jellyfin container.

Browsers (Jellyfin Web) default to "Prefer fMP4-HLS Media Container", which direct-streams HEVC to the browser. Browsers that claim but can't actually decode HEVC 10-bit stall after a few seconds. The setting is per-browser localStorage; there is no server-wide toggle.

Verify capability claims with `docker exec jellyfin /usr/lib/jellyfin-ffmpeg/vainfo --display drm --device /dev/dri/renderD128` rather than assuming.

**Media data layout (`/mnt/void/data`):** Shared filesystem contract across the media stack:
- `/mnt/void/data/torrents` — qBittorrent download target
- `/mnt/void/data/media` — final library, consumed by Jellyfin and Bazarr
- The *arrs bind-mount `/mnt/void/data` (the parent) at `/data`; qBittorrent mounts only `/mnt/void/data/torrents` at `/data/torrents`. The container paths match, so the *arrs hardlink between `torrents/` and `media/` instead of copying. Don't change the *arr mounts to point at a subtree — that silently doubles disk usage.
- Janitorr creates "Leaving Soon" symlinks under `media/leaving-soon/` that point at the *arr paths (`/data/media/...`). Jellyfin therefore mounts the media twice: `/media` (its libraries) and a read-only `/data/media` so those symlink targets resolve. Removing the second mount breaks the Leaving Soon libraries.

**Host-specific paths:** A few bind mounts assume this is `matrix`:
- `/mnt/void/.beszel` (beszel-agent disk monitoring)
- `/dev/dri/renderD128` + `group_add: '105'` (Jellyfin GPU/render group)
- `/home/debian/homelab` (Dockge stack directory)

These need adjusting if the repo moves to a different host.

**Secrets and data:** `.env` files (gitignored) hold secrets referenced as `${VAR}`. Persistent data lives in `<service>/container_data/` (gitignored).

**Backups:** None currently. `xutils/backup.sh` is a WIP scaffold and not in use — `container_data/` and `acme.json` are not snapshotted anywhere. To-do.

**Inbound ports:** No port forwarding is configured on the router — nothing on this host is reachable from the public internet, only from LAN and Tailscale. Only Traefik's `80` and `443` are published to the host, plus `scanopy-server` on `127.0.0.1:60072` (loopback only) because `scanopy-daemon` runs with `network_mode: host` and reaches the server via `localhost:60072`.

**Tribal-knowledge Traefik middlewares** (easy to break inadvertently when editing labels):
- `qbittorrent` — clears `Origin` and `Referer` headers and sets `passhostheader: false`. qBit's CSRF check rejects requests where `Origin` doesn't match the bind address; clearing them is what makes the WebUI work behind Traefik.
- `jellyfin` — full HSTS hardening (preload, includeSubdomains, 10-year max-age), `frameDeny`, `contentTypeNosniff`, plus a custom `X-Robots-Tag: noindex,nofollow,...` so the public Jellyfin URL doesn't show up in search engines.
- `whoami` — injects `Connection: Upgrade` / `Upgrade: websocket` headers (it's a WebSocket demo).

**Observability:** Beszel (host metrics + agent), Uptime Kuma (HTTP uptime), Dozzle (live container logs), Homepage (dashboard). Start here when something looks wrong.

## Compose File Conventions

- Volumes use the expanded bind-mount format with explicit `read_only` keys. Host paths for persistent data follow `./container_data/<container_name>/<basename>` where `<basename>` matches the last component of the container-side path (e.g. `/app/config` → `./container_data/homepage/config`, `/beszel_data` → `./container_data/beszel/beszel_data`).
- `environment` and `labels` sections use dict format (not list), sorted alphabetically.
- `restart: unless-stopped` on all services.
- LinuxServer.io images use `PUID: 1000`, `PGID: 1000`, `TZ: Asia/Kolkata`.
- Service keys within each service block are sorted alphabetically. `xutils/fmt-compose.py` normalizes this (sorted keys/env/labels, expanded volumes); with no arguments it rewrites every `compose.yaml` in the repo, or pass specific paths. It is not part of any automated workflow.
