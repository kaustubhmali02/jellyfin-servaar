# 🍿✨ The Ultimate Self-Hosted Media & \*arr Stack (`compose.yaml`) 🚀

Welcome to the engine room of your personal streaming empire! 🏰🎬 This repository orchestrates a **16-container Docker Compose powerhouse** that automates media discovery, downloading, subtitle hunting, GPU-accelerated streaming, watch-time analytics, and server management. 

Every media manager chats happily over a dedicated static-subnet bridge network (`servarrnetwork` on `172.39.0.0/24` 🌐), while all your torrenting and indexer traffic stays locked inside an encrypted VPN bunker (`gluetun` 🛡️🔒).

## 🎢 How the Magic Happens (End-to-End Workflow)

* 🛎️ **Step 1 — Order Up! (Request & Discovery)**: Users browse movies and shows in **Seerr** (`:5055`), which checks **Jellyfin** (`:8096`) so nobody requests what you already own, then hands the shopping list to **Sonarr** (`:8989`) or **Radarr** (`:7878`).
* 🕵️‍♂️ **Step 2 — The Treasure Hunt (Indexing & Cloudflare Bypass)**: **Sonarr**, **Radarr**, and **Lidarr** (`:8686`) ask **Prowlarr** (`:9696`) to scour torrent trackers and Usenet indexers. If a site throws up a stubborn Cloudflare wall, **FlareSolverr** (`:8191`) swoops in to solve the puzzle! 🧩
* 🛡️ **Step 3 — Stealth Mode & Self-Healing Downloads**: **qBittorrent** (`:8080`) and **NZBGet** (`:6789`) pull down files strictly through **Gluetun**'s VPN tunnel. If the VPN hiccups and **qBittorrent** stalls out, **Deunhealth** acts as the on-call paramedic 🚑—spotting the failed healthcheck and restarting **qBittorrent** automatically.
* 🎞️ **Step 4 — Instant Organization, Subtitles & Popcorn Time**: Completed files in `${DATA}:/data` are hardlinked in a flash ⚡ into clean library folders by the `*arr` crew. **Bazarr** (`:6767`) grabs matching subtitles 💬, and **Jellyfin** streams the final product with buttery-smooth NVIDIA GPU hardware transcoding 🎮🔥.
* 📊 **Step 5 — Nerd Stats & Mission Control**: **Jellystat** (`:3000`) and its PostgreSQL sidekick (**Jellystat-DB** 🐘) crunch your viewing numbers, **Homarr** (`:7575`) ties everything together on a sleek dashboard 🖥️, and **Portainer** (`:9443`) gives you total command over your Docker containers 🐳.

## 🧩 Meet the 16 Docker Services (And Their Unique Variables!)

Here is the point-by-point roll call of all 16 containers in `compose.yaml`. To keep things tidy and avoid déjà vu 🔁, environment and Compose variables are showcased with examples under the **first service that introduces them**, while later services simply note which shared variables they reuse!

### 1. 🛡️ `gluetun` (`qmcgaw/gluetun`) — *The VPN Bouncer*
* 🎯 **What It Does**: Acts as the armored VPN gateway and firewall for every container that talks to trackers or download peers.
* 🤝 **How It Plays With Others**: Sits on `servarrnetwork` with a static IP and lends its entire network stack (`network_mode: service:gluetun`) to `qbittorrent`, `nzbget`, `prowlarr`, and `flaresolverr`. Because those four ride piggyback on Gluetun, `gluetun` publishes all of their ports (`8080`, `6881`, `6789`, `9696`, `8191`) plus its own control port (`8000` for `homarr`). It pings `www.google.com` every 20 seconds 💓—if the VPN isn't healthy, none of the downloaders are allowed out of bed!
* ⚙️ **Under the Hood**: Uses `NET_ADMIN` capability and `/dev/net/tun` to build the VPN tunnel, pulls secrets from `.env`, and stores state in `./gluetun:/gluetun`.
* 🔑 **New Variables Unlocked**:
  * `SET_IP_GLUETUN=172.39.0.2` — Static IPv4 address for Gluetun on `servarrnetwork` 🌐.
  * *(💡 Bonus: `gluetun` also reads your VPN credentials straight from `.env` via `env_file: .env`, like `VPN_SERVICE_PROVIDER=airvpn`, `VPN_TYPE=wireguard`, and `WIREGUARD_PRIVATE_KEY=...`.)*

### 2. 🌊 `qbittorrent` (`lscr.io/linuxserver/qbittorrent:latest`) — *The Torrent Workhorse*
* 🎯 **What It Does**: Downloads magnet links and `.torrent` files ordered by Sonarr, Radarr, and Lidarr.
* 🤝 **How It Plays With Others**: Tunnels 100% of its traffic through `gluetun` (`network_mode: service:gluetun`) and waits patiently for `gluetun` to pass its healthcheck (`depends_on` with `service_healthy` and `restart: true`) 🔗. It wears the `deunhealth.restart.on.unhealthy=true` badge so `deunhealth` can reboot it if a VPN drop leaves it stranded, and writes finished files straight to `${DATA}:/data` 📂.
* 🔑 **New Variables Unlocked**:
  * `PUID=1000` — Linux User ID so downloaded files belong to you, not `root` 👤.
  * `PGID=1000` — Linux Group ID for clean shared folder permissions 👥.
  * `TZ=Asia/Kolkata` — Keeps container clocks and schedules in your local timezone 🕒.
  * `WEBUI_PORT=8080` — Matches the Web UI port opened on `gluetun` 🖥️.
  * `DATA=/mnt/storage/data` — The master host path mounted to `/data` for downloads and media 💾.
  * `FIREWALL_VPN_INPUT_PORTS=49152` *(optional, commented as `TORRENTING_PORT=${FIREWALL_VPN_INPUT_PORTS}`)* — Port-forwarding number from your VPN provider for faster seeding 🚀.

### 3. 🚑 `deunhealth` (`qmcgaw/deunhealth`) — *The Container Paramedic*
* 🎯 **What It Does**: Watches labeled containers like a hawk and restarts them the moment their healthcheck goes red 🚨.
* 🤝 **How It Plays With Others**: Has zero network access (`network_mode: "none"` 📵) and talks directly to the Docker daemon through `/var/run/docker.sock`. Whenever `qbittorrent` stalls after a VPN hiccup and fails its ping test, `deunhealth` spots the `deunhealth.restart.on.unhealthy=true` label and hits the reset button 🔄!
* 🔑 **New Variables Unlocked** *(reuses `TZ` 🕒)*:
  * `LOG_LEVEL=info` — Controls how chatty the logs are (`debug`, `info`, `warn`, `error`) 📝.
  * `HEALTH_SERVER_ADDRESS=127.0.0.1:9999` — Internal loopback address and port for Deunhealth's own status check 🩺.

### 4. 📦 `nzbget` (`lscr.io/linuxserver/nzbget:latest`) — *The Usenet Speedster*
* 🎯 **What It Does**: Downloads, verifies, and unpacks Usenet `.nzb` archives at lightning speed ⚡.
* 🤝 **How It Plays With Others**: Shares `gluetun`'s VPN network (`network_mode: service:gluetun`, Web UI on port `6789`) and waits for `gluetun` to be healthy (`restart: true`) 🛡️. Drops unpacked files right into `${DATA}:/data` so Sonarr, Radarr, and Lidarr can scoop them up 📂.
* 🔑 **New Variables Unlocked**:
  * *✨ None! Happily reuses `PUID`, `PGID`, `TZ`, and `DATA`.*

### 5. 🕵️ `prowlarr` (`lscr.io/linuxserver/prowlarr:latest`) — *The Indexer Mastermind*
* 🎯 **What It Does**: Manages all your torrent trackers and Usenet indexers in one central hub 🗂️.
* 🤝 **How It Plays With Others**: Hides behind `gluetun`'s VPN (`network_mode: service:gluetun`, Web UI on port `9696`) and waits for a healthy tunnel (`restart: true`) so ISP-blocked trackers are never an issue 🕶️. Automatically pushes indexer configs to `sonarr`, `radarr`, and `lidarr`, and calls on `flaresolverr` whenever Cloudflare gets in the way 🤝.
* 🔑 **New Variables Unlocked**:
  * *✨ None! Reuses `PUID`, `PGID`, and `TZ`.*

### 6. 🔥 `flaresolverr` (`ghcr.io/flaresolverr/flaresolverr:latest`) — *The Cloudflare Whisperer*
* 🎯 **What It Does**: Spins up a stealthy headless browser proxy to breeze past Cloudflare and DDoS-GUARD challenges 🛡️🔓.
* 🤝 **How It Plays With Others**: Lives inside `gluetun`'s network namespace (`network_mode: service:gluetun`, port `8191`) and depends on a healthy `gluetun` tunnel. Because `prowlarr` and `flaresolverr` share the exact same VPN container network, `prowlarr` can ring up FlareSolverr right at `http://localhost:8191` 📞!
* 🔑 **New Variables Unlocked** *(reuses `TZ` 🕒; upgrades `LOG_LEVEL` with a `.env` default fallback)*:
  * `LOG_LEVEL=info` *(via `${LOG_LEVEL:-info}`)* — Sets log verbosity, falling back to `info` if left blank in `.env` 📋.
  * `LOG_HTML=false` *(via `${LOG_HTML:-false}`)* — Whether to dump raw HTML into logs for troubleshooting (`true`/`false`) 🧪.
  * `CAPTCHA_SOLVER=none` *(via `${CAPTCHA_SOLVER:-none}`)* — Optional captcha-solving plugin setting 🤖.

### 7. 📺 `sonarr` (`lscr.io/linuxserver/sonarr:latest`) — *The TV Show Showrunner*
* 🎯 **What It Does**: Tracks, grabs, renames, and organizes all your TV series and anime episodes 🍿📺.
* 🤝 **How It Plays With Others**: Lives on `servarrnetwork` (`:8989`). Takes show orders from `seerr`, searches indexers synced from `prowlarr`, dispatches downloads to `qbittorrent` or `nzbget` (via `gluetun`), hardlinks finished episodes in `${DATA}:/data` for `jellyfin`, and pings `bazarr` to grab subtitles 💬.
* 🔑 **New Variables Unlocked** *(reuses `PUID`, `PGID`, `TZ`, and `DATA`)*:
  * `SET_IP_SONARR=172.39.0.3` — Static IPv4 address for Sonarr on `servarrnetwork` 🌐.

### 8. 🎬 `radarr` (`lscr.io/linuxserver/radarr:latest`) — *The Movie Director*
* 🎯 **What It Does**: Curates, downloads, upgrades, and organizes your entire movie collection 🎥🎞️.
* 🤝 **How It Plays With Others**: Lives on `servarrnetwork` (`:7878`). Receives movie requests from `seerr`, searches via `prowlarr`, sends jobs to `qbittorrent`/`nzbget`, organizes films inside `${DATA}:/data` for `jellyfin`, and teams up with `bazarr` for movie subtitles 💬.
* 🔑 **New Variables Unlocked** *(reuses `PUID`, `PGID`, `TZ`, and `DATA`)*:
  * `SET_IP_RADARR=172.39.0.4` — Static IPv4 address for Radarr on `servarrnetwork` 🌐.

### 9. 🎵 `lidarr` (`lscr.io/linuxserver/lidarr:latest`) — *The Music DJ*
* 🎯 **What It Does**: Automatically follows artists, grabs albums, and keeps your music library tagged and spotless 🎧🎶.
* 🤝 **How It Plays With Others**: Lives on `servarrnetwork` (`:8686`). Searches music indexers provided by `prowlarr`, sends downloads to `qbittorrent` or `nzbget`, and organizes audio tracks inside `${DATA}:/data` for `jellyfin` 📻.
* 🔑 **New Variables Unlocked** *(reuses `PUID`, `PGID`, `TZ`, and `DATA`)*:
  * `SET_IP_LIDARR=172.39.0.5` — Static IPv4 address for Lidarr on `servarrnetwork` 🌐.

### 10. 💬 `bazarr` (`lscr.io/linuxserver/bazarr:latest`) — *The Polyglot Subtitle Hunter*
* 🎯 **What It Does**: Automatically finds, downloads, and syncs missing subtitles in your favorite languages 🌍📝.
* 🤝 **How It Plays With Others**: Lives on `servarrnetwork` (`:6767`) and shares `${DATA}:/data`. Talks directly to `sonarr` and `radarr` APIs to see what episodes or movies just landed, then drops `.srt` files right next to the video files so `jellyfin` picks them up instantly ✨.
* 🔑 **New Variables Unlocked** *(reuses `PUID`, `PGID`, `TZ`, and `DATA`)*:
  * `SET_IP_BAZARR=172.39.0.6` — Static IPv4 address for Bazarr on `servarrnetwork` 🌐.

### 11. 🛎️ `seerr` (`ghcr.io/seerr-team/seerr:latest`) — *The Request Concierge*
* 🎯 **What It Does**: Gives friends and family a gorgeous Netflix-style portal to discover and request movies and TV shows 🌟.
* 🤝 **How It Plays With Others**: Lives on `servarrnetwork` (`:5055`) and waits for `jellyfin` to report healthy (`depends_on: jellyfin: condition: service_healthy`) before booting up 🚦. Logs users in with their `jellyfin` accounts, checks what is already in the library, and shoots approved requests over to `sonarr` and `radarr` 🚀.
* 🔑 **New Variables Unlocked** *(reuses `LOG_LEVEL=debug`, `PUID`, `PGID`, and `TZ`)*:
  * `SET_IP_SEERR=172.39.0.7` — Static IPv4 address for Seerr on `servarrnetwork` 🌐.

### 12. 🍿 `jellyfin` (`lscr.io/linuxserver/jellyfin:latest`) — *The Home Theater Star*
* 🎯 **What It Does**: Streams your movies, TV shows, and music to any screen with zero subscription fees and full NVIDIA GPU hardware transcoding 📺🔥.
* 🤝 **How It Plays With Others**: Anchors `servarrnetwork` on HTTP port `8096` (plus UDP discovery ports `7359/udp` and `1900/udp` 📡). Reads all organized media from `${DATA}:/data`, taps into your host GPU (`nvidia.com/gpu=all`), acts as the startup dependency for `seerr`, and feeds live playback data to `jellystat` 📈.
* 🔑 **New Variables Unlocked** *(reuses `PUID`, `PGID`, `TZ`, and `DATA`)*:
  * `NVIDIA_VISIBLE_DEVICES=all` — Hands your NVIDIA GPU(s) over to Jellyfin for hardware transcoding 🎮.
  * `NVIDIA_DRIVER_CAPABILITIES=compute,video,utility` — Activates NVENC/NVDEC video acceleration features ⚡.
  * `SET_IP_JELLYFIN=172.39.0.8` — Static IPv4 address for Jellyfin on `servarrnetwork` 🌐.
  * `JELLYFIN_PublishedServerUrl=http://10.0.0.101` *(optional, commented in `compose.yaml`)* — Custom LAN URL broadcast to local Jellyfin clients 📢.

### 13. 🐘 `jellystat-db` (`postgres:18.1`) — *The Stats Vault*
* 🎯 **What It Does**: Runs a dedicated PostgreSQL 18.1 database (`1gb` shared memory) to store all of Jellystat's watch history and metrics 🗄️.
* 🤝 **How It Plays With Others**: Sits on `servarrnetwork`, saves data in `./jellystat/postgres-data:/var/lib/postgresql`, and runs a `pg_isready` healthcheck every 10 seconds so `jellystat` knows exactly when the database doors are open 🚪✅.
* 🔑 **New Variables Unlocked**:
  * `JELLYSTAT_POSTGRES_USER=jellystat_admin` *(mapped to `POSTGRES_USER`)* — Database username 👤.
  * `JELLYSTAT_POSTGRES_PASSWORD=SuperSecretPgPass123!` *(mapped to `POSTGRES_PASSWORD`)* — Database password 🔐.
  * `SET_IP_JELLYSTAT_DB=172.39.0.9` — Static IPv4 address for the database on `servarrnetwork` 🌐.

### 14. 📊 `jellystat` (`cyfershepard/jellystat:latest`) — *The Data Cruncher*
* 🎯 **What It Does**: Turns your Jellyfin activity into rich charts showing who watched what, when, and on which device 📈🕵️‍♀️.
* 🤝 **How It Plays With Others**: Runs on `servarrnetwork` (`:3000`). Waits for `jellystat-db` to pass its healthcheck (`depends_on: jellystat-db: condition: service_healthy`), stores records in `jellystat-db` (`:5432`), and polls `jellyfin` (`:8096`) for live playback sessions 📡.
* 🔑 **New Variables Unlocked** *(reuses `JELLYSTAT_POSTGRES_USER`, `JELLYSTAT_POSTGRES_PASSWORD`, and `TZ`)*:
  * `POSTGRES_IP=jellystat-db` — Internal Docker DNS name pointing straight to the database container 🧭.
  * `POSTGRES_PORT=5432` — Standard PostgreSQL port 🔌.
  * `JELLYSTAT_JWT_SECRET=9f8e7d6c5b4a3f2e1d0c9b8a7f6e5d4c3b2a1f0e` *(mapped to `JWT_SECRET`)* — Secret key used to sign login session tokens 🔏.
  * `SET_IP_JELLYSTAT=172.39.0.10` — Static IPv4 address for Jellystat on `servarrnetwork` 🌐.

### 15. 🐳 `portainer` (`portainer/portainer-ce:lts`) — *The Fleet Commander*
* 🎯 **What It Does**: Gives you a powerful web UI (`:9443`) to inspect logs, restart containers, manage volumes, and monitor resource usage 🎛️🚢.
* 🤝 **How It Plays With Others**: Runs with `privileged: true` on `servarrnetwork` and hooks into `/var/run/docker.sock:/var/run/docker.sock` to oversee all 16 containers in your homelab 🔭.
* 🔑 **New Variables Unlocked**:
  * `SET_IP_PORTAINER=172.39.0.11` — Static IPv4 address for Portainer on `servarrnetwork` 🌐.

### 16. 🖥️ `homarr` (`ghcr.io/homarr-labs/homarr:latest`) — *The Mission Control Dashboard*
* 🎯 **What It Does**: Wraps your entire server into a customizable start page (`:7575`) with live widgets for downloads, upcoming episodes, and active streams 🏠✨.
* 🤝 **How It Plays With Others**: Sits on `servarrnetwork`, reads `/var/run/docker.sock` for live container status badges 🟢, queries `gluetun` (`:8000` control server) for VPN status, and connects to your `*arr` apps, `seerr`, `qbittorrent`, `nzbget`, and `jellyfin` APIs 🔗.
* 🔑 **New Variables Unlocked**:
  * `SECRET_ENCRYPTION_KEY=a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90` — 64-character hex key used to lock down API keys saved in Homarr 🔐.
  * `SET_IP_HOMARR=172.39.0.12` — Static IPv4 address for Homarr on `servarrnetwork` 🌐.

## 🕸️ Deep Dive: How the Services Connect & Cooperate

### 1. 🌐 Three-Tier Network Architecture
Your containers are organized into three smart network zones so privacy and speed never clash:

| Network Zone | Who Lives Here | How It Works |
| :--- | :--- | :--- |
| **🌉 `servarrnetwork` (`172.39.0.0/24`)** | `gluetun`, `sonarr`, `radarr`, `lidarr`, `bazarr`, `seerr`, `jellyfin`, `jellystat-db`, `jellystat`, `portainer`, `homarr` | Every service gets its own predictable static IP (`${SET_IP_*}`). Apps talk to each other at LAN speed over this private bridge! ⚡ |
| **🛡️ `network_mode: service:gluetun`** | `qbittorrent`, `nzbget`, `prowlarr`, `flaresolverr` | These four containers literally live *inside* `gluetun`'s network stack. If the VPN drops, their internet vanishes instantly (built-in killswitch 🔒). All their ports (`8080`, `6881`, `6789`, `9696`, `8191`) are opened on `gluetun`. |
| **📵 `network_mode: "none"`** | `deunhealth` | Zero network access needed! It only talks to the local Docker socket (`/var/run/docker.sock`) to reboot unhealthy containers. |

### 2. 🚦 Domino-Effect Startup & Self-Healing Chains
Nobody likes race conditions on boot! Here is how the stack keeps startups orderly and self-healing:
* 🛡️ **The VPN Gatekeeper (`gluetun` ➡️ Downloaders & Indexers)**: `qbittorrent`, `nzbget`, `prowlarr`, and `flaresolverr` all use `depends_on: gluetun` (`condition: service_healthy`, `restart: true`). They wait until `gluetun` can ping Google before launching, and if `gluetun` ever restarts, Docker automatically restarts all four so their shared network namespace stays intact 🔄.
* 🚑 **The Auto-Revive Loop (`deunhealth` ➡️ `qbittorrent`)**: If a temporary VPN blip causes `qbittorrent` to fail its 60-second ping check, `deunhealth` spots the `deunhealth.restart.on.unhealthy=true` label via `/var/run/docker.sock` and restarts `qbittorrent` hands-free 🙌.
* 🍿 **The Streaming Gatekeeper (`jellyfin` ➡️ `seerr`)**: `seerr` waits for `jellyfin`'s `/health` check to pass before starting, guaranteeing your media library is ready to query ✅.
* 🐘 **The Database Gatekeeper (`jellystat-db` ➡️ `jellystat`)**: `jellystat` waits until PostgreSQL passes its `pg_isready` check before booting up 📊.

### 3. 💾 Shared Storage & Hardlink Superpowers
* ⚡ **One Mount to Rule Them All (`${DATA}:/data`)**: `qbittorrent`, `nzbget`, `sonarr`, `radarr`, `lidarr`, `bazarr`, and `jellyfin` all mount the exact same `${DATA}:/data` path with matching `PUID` and `PGID` permissions. Because Docker sees `/data` as a single unified filesystem, importing a 50 GB 4K movie from `/data/downloads` into `/data/media/movies` uses **instant hardlinks**—taking 0 milliseconds and 0 extra bytes of disk space while seeding continues! 🎉
* 🔌 **Docker Socket Trio (`/var/run/docker.sock`)**: Shared by `deunhealth` (auto-healing), `portainer` (container management), and `homarr` (live status badges).

## 📋 Master Cheat Sheet: Unique Variables (Zero Repeats!)

| First Seen In | Variable Name | Type | Example Value | What It Does |
| :--- | :--- | :--- | :--- | :--- |
| 🛡️ `gluetun` | `SET_IP_GLUETUN` | `.env` | `172.39.0.2` | Static IPv4 for `gluetun` on `servarrnetwork` |
| 🌊 `qbittorrent` | `PUID` | `.env` | `1000` | User ID for file & process ownership |
| 🌊 `qbittorrent` | `PGID` | `.env` | `1000` | Group ID for file & process ownership |
| 🌊 `qbittorrent` | `TZ` | `.env` | `Asia/Kolkata` | System timezone across the stack |
| 🌊 `qbittorrent` | `WEBUI_PORT` | Hardcoded | `8080` | Internal port for qBittorrent's Web UI |
| 🌊 `qbittorrent` | `DATA` | `.env` | `/mnt/storage/data` | Master host folder mounted to `/data` |
| 🚑 `deunhealth` | `LOG_LEVEL` | Hardcoded / `.env` | `info` | Log verbosity (`info`, `debug`, `warn`, `error`) |
| 🚑 `deunhealth` | `HEALTH_SERVER_ADDRESS` | Hardcoded | `127.0.0.1:9999` | Internal health check bind address |
| 🔥 `flaresolverr` | `LOG_HTML` | `.env` + Default | `false` | Log full HTML responses (`${LOG_HTML:-false}`) |
| 🔥 `flaresolverr` | `CAPTCHA_SOLVER` | `.env` + Default | `none` | Captcha solver plugin (`${CAPTCHA_SOLVER:-none}`) |
| 📺 `sonarr` | `SET_IP_SONARR` | `.env` | `172.39.0.3` | Static IPv4 for `sonarr` on `servarrnetwork` |
| 🎬 `radarr` | `SET_IP_RADARR` | `.env` | `172.39.0.4` | Static IPv4 for `radarr` on `servarrnetwork` |
| 🎵 `lidarr` | `SET_IP_LIDARR` | `.env` | `172.39.0.5` | Static IPv4 for `lidarr` on `servarrnetwork` |
| 💬 `bazarr` | `SET_IP_BAZARR` | `.env` | `172.39.0.6` | Static IPv4 for `bazarr` on `servarrnetwork` |
| 🛎️ `seerr` | `SET_IP_SEERR` | `.env` | `172.39.0.7` | Static IPv4 for `seerr` on `servarrnetwork` |
| 🍿 `jellyfin` | `NVIDIA_VISIBLE_DEVICES` | `.env` | `all` | Exposes host NVIDIA GPUs for transcoding |
| 🍿 `jellyfin` | `NVIDIA_DRIVER_CAPABILITIES` | `.env` | `compute,video,utility` | Unlocks NVENC/NVDEC driver capabilities |
| 🍿 `jellyfin` | `SET_IP_JELLYFIN` | `.env` | `172.39.0.8` | Static IPv4 for `jellyfin` on `servarrnetwork` |
| 🐘 `jellystat-db` | `JELLYSTAT_POSTGRES_USER` | `.env` | `jellystat_admin` | PostgreSQL database username |
| 🐘 `jellystat-db` | `JELLYSTAT_POSTGRES_PASSWORD` | `.env` | `SuperSecretPgPass123!` | PostgreSQL database password |
| 🐘 `jellystat-db` | `SET_IP_JELLYSTAT_DB` | `.env` | `172.39.0.9` | Static IPv4 for `jellystat-db` on `servarrnetwork` |
| 📊 `jellystat` | `POSTGRES_IP` | Hardcoded | `jellystat-db` | Internal hostname of the Postgres container |
| 📊 `jellystat` | `POSTGRES_PORT` | Hardcoded | `5432` | Internal TCP port of the Postgres container |
| 📊 `jellystat` | `JELLYSTAT_JWT_SECRET` | `.env` | `9f8e7d6c5b4a3f2e1d0c9b8a7f6e5d4c` | Secret key for signing Jellystat auth tokens |
| 📊 `jellystat` | `SET_IP_JELLYSTAT` | `.env` | `172.39.0.10` | Static IPv4 for `jellystat` on `servarrnetwork` |
| 🐳 `portainer` | `SET_IP_PORTAINER` | `.env` | `172.39.0.11` | Static IPv4 for `portainer` on `servarrnetwork` |
| 🖥️ `homarr` | `SECRET_ENCRYPTION_KEY` | `.env` | `a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90` | 64-char hex key for Homarr credential encryption |
| 🖥️ `homarr` | `SET_IP_HOMARR` | `.env` | `172.39.0.12` | Static IPv4 for `homarr` on `servarrnetwork` |

## 🛠️ Copy-Paste Ready `.env` Template

Drop this into a `.env` file right next to `compose.yaml`, tweak your paths and passwords, and fire it up with `docker compose up -d`! 🚀

```dotenv
# 👤 Global User, Timezone & Storage Paths
PUID=1000
PGID=1000
TZ=Asia/Kolkata
DATA=/mnt/storage/data

# 🔥 FlareSolverr Optional Overrides
LOG_LEVEL=info
LOG_HTML=false
CAPTCHA_SOLVER=none

# 🎮 Jellyfin NVIDIA Hardware Acceleration
NVIDIA_VISIBLE_DEVICES=all
NVIDIA_DRIVER_CAPABILITIES=compute,video,utility

# 📊 Jellystat & PostgreSQL Credentials
JELLYSTAT_POSTGRES_USER=jellystat_admin
JELLYSTAT_POSTGRES_PASSWORD=SuperSecretPgPass123!
JELLYSTAT_JWT_SECRET=9f8e7d6c5b4a3f2e1d0c9b8a7f6e5d4c3b2a1f0e

# 🖥️ Homarr Encryption Key (generate yours with: openssl rand -hex 32)
SECRET_ENCRYPTION_KEY=a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90

# 🌐 Static IPv4 Assignments on servarrnetwork (172.39.0.0/24)
SET_IP_GLUETUN=172.39.0.2
SET_IP_SONARR=172.39.0.3
SET_IP_RADARR=172.39.0.4
SET_IP_LIDARR=172.39.0.5
SET_IP_BAZARR=172.39.0.6
SET_IP_SEERR=172.39.0.7
SET_IP_JELLYFIN=172.39.0.8
SET_IP_JELLYSTAT_DB=172.39.0.9
SET_IP_JELLYSTAT=172.39.0.10
SET_IP_PORTAINER=172.39.0.11
SET_IP_HOMARR=172.39.0.12

# 🛡️ Gluetun VPN Configuration Example (AirVPN / WireGuard)
VPN_SERVICE_PROVIDER=airvpn
VPN_TYPE=wireguard
WIREGUARD_PRIVATE_KEY=your_wireguard_private_key_here
WIREGUARD_PRESHARED_KEY=your_wireguard_preshared_key_here
WIREGUARD_ADDRESSES=10.x.x.x/32
FIREWALL_VPN_INPUT_PORTS=49152
```