# Ugreen NAS (DXP2800) Arr Stack Setup Guide
---
This guide covers setting up a fully containerized Arr stack with a secure WireGuard VPN connection via Gluetun on a Ugreen NAS running UGOS Pro.
The steps in this guide are based on a YouTube video by Digital Aloha ([watch it here](https://www.youtube.com/watch?v=Z2yoJpKl59g&t=468s)). 
I highly recommend watching his video on how to set up the Arr stack on a Ugreen NAS, as it is very helpful and informative. 
In this guide, I have made a few minor alterations for myself, such as using WireGuard instead of OpenVPN because it's faster, at least for me. 
The Jellyfin section is something I troubleshot myself in order to get QuickSync working in Docker on the NAS.

## Step 1: Storage Setup & Folder Structure

### Folder Types in UGOS Pro
* **`arr-stack-data` (Shared Folder):** Must be created as a **Shared Folder** in **Control Panel > Shared Folder** (or **Files** app). This allows network access and root path mapping across containers.
* **Subfolders & Docker Config Folder:** Created as standard subfolders inside `arr-stack-data` or under the default `docker` shared folder.

### Required Directory Structure
Create your subfolders exactly according to this hierarchy:

```text
/volume1/
├── docker/
│   └── arr-stack/                <-- Stores .env and docker-compose.yaml
│       ├── gluetun/             <-- App configs / data directory
│       ├── sabnzbd/             <-- App configs
│       ├── radarr/
│       ├── sonarr/
│       ├── prowlarr/
│       ├── bazarr/
│       └── jellyfin/
└── arr-stack-data/               <-- SHARED FOLDER
    ├── media/
    │   ├── movies/              <-- Managed library files
    │   └── tv/
    ├── torrents/
    │   ├── completed/           <-- Torrent downloads (if used)
    │   └── incomplete/
    └── usenet/
        ├── completed/           <-- Usenet downloads
        │   ├── movies/
        │   └── tv/
        └── incomplete/
```

---

## Step 2: Create a Dedicated User & Get UID/GID

To run your containers safely with proper file ownership, you should create a dedicated non-admin user account in UGOS:

1. Open **Control Panel** > **User Management** > **User** and create a new user (e.g., `arr-user`). 
   > **Note:** You **do not** need to remember or keep track of the login credentials for this user, as it is strictly used as a backend service account for permission management.
2. Under **Permissions & Settings**, give this user **Read/Write** permissions for both the **`docker`** and **`arr-stack-data`** shared folders.
3. Open the NAS terminal tool (or SSH) and find the user's **UID** and **GID** by running:
   ```bash
   id your-username
   ```
   * *Example output:* `uid=1001(arr-user) gid=100(users) groups=100(users)...`
   * Note down your specific `uid` and `gid` numbers, as you will use them for `PUID` and `PGID` in your `.env` file.

### Apply Permissions via Terminal
Run the following commands in your NAS terminal to set up the directories and enforce the correct ownership and permissions:

```bash
# 1. Create directory structure if not already made via GUI
sudo mkdir -p /volume1/docker/arr-stack/gluetun
sudo mkdir -p /volume1/docker/arr-stack/sabnzbd
sudo mkdir -p /volume1/docker/arr-stack/radarr
sudo mkdir -p /volume1/docker/arr-stack/sonarr
sudo mkdir -p /volume1/docker/arr-stack/prowlarr
sudo mkdir -p /volume1/docker/arr-stack/bazarr
sudo mkdir -p /volume1/docker/arr-stack/jellyfin
sudo mkdir -p /volume1/arr-stack-data/media/movies
sudo mkdir -p /volume1/arr-stack-data/media/tv
sudo mkdir -p /volume1/arr-stack-data/torrents/completed
sudo mkdir -p /volume1/arr-stack-data/torrents/incomplete
sudo mkdir -p /volume1/arr-stack-data/usenet/completed/movies
sudo mkdir -p /volume1/arr-stack-data/usenet/completed/tv
sudo mkdir -p /volume1/arr-stack-data/usenet/incomplete

# 2. Assign ownership to your dedicated user and standard group
sudo chown -R your-username:users /volume1/arr-stack-data /volume1/docker/arr-stack/

# 3. Apply standard directory permissions
sudo chmod -R a=,a+rX,u+w,g+w /volume1/arr-stack-data /volume1/docker/arr-stack/

# 4. Grant write access to usenet downloads
sudo chmod -R o+w /volume1/arr-stack-data/usenet
```

---

## Step 3: Create the `.env` File

Place a `.env` file in `/volume1/docker/arr-stack/.env` using the UID and GID found in Step 2:
You can download the TextEdit app from the Ugreen App Center on your NAS, create and save a .env.txt file, and then go to the folder where you saved it and rename the file from .env.txt to .env

```env
# VPN Configuration (Replace with your actual WireGuard private key)
WIREGUARD_PRIVATE_KEY=your_wireguard_private_key_here

# User and System Identifiers (Update PUID and PGID with values from 'id your-username')
PUID=1001
PGID=100
TZ=Europe/Amsterdam

# Directory Paths
DOCKERCONFDIR=/volume1/docker/arr-stack
DOCKERSTORAGEDIR=/volume1/arr-stack-data

# Docker Logging Options
DOCKERLOGGING_MAXFILE=10
DOCKERLOGGING_MAXSIZE=200k
```

---

## Step 4: Create the `docker-compose.yaml` File

Create `docker-compose.yaml` inside `/volume1/docker/arr-stack/`:

```yaml
services:
  gluetun:
    image: qmcgaw/gluetun:v3
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    environment:
      - VPN_SERVICE_PROVIDER=nordvpn
      - VPN_TYPE=wireguard
      - WIREGUARD_PRIVATE_KEY=${WIREGUARD_PRIVATE_KEY}
      - SERVER_CITIES=Amsterdam
    ports:
      - 8080:8080 # SABnzbd Web UI
    restart: unless-stopped

  sabnzbd:
    image: lscr.io/linuxserver/sabnzbd:latest
    container_name: sabnzbd
    network_mode: "service:gluetun"
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${DOCKERCONFDIR}/sabnzbd:/config
      - ${DOCKERSTORAGEDIR}/usenet:/data/usenet
    depends_on:
      gluetun:
        condition: service_started
    restart: unless-stopped

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    ports:
      - 7878:7878
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${DOCKERCONFDIR}/radarr:/config
      - ${DOCKERSTORAGEDIR}:/data
    restart: unless-stopped

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    ports:
      - 8989:8989
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${DOCKERCONFDIR}/sonarr:/config
      - ${DOCKERSTORAGEDIR}:/data
    restart: unless-stopped

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    ports:
      - 9696:9696
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${DOCKERCONFDIR}/prowlarr:/config
    restart: unless-stopped

  bazarr:
    image: lscr.io/linuxserver/bazarr:latest
    container_name: bazarr
    ports:
      - 6767:6767
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${DOCKERCONFDIR}/bazarr:/config
      - ${DOCKERSTORAGEDIR}/media:/data/media
    restart: unless-stopped

  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    network_mode: "host"
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - DOCKER_MODS=linuxserver/mods:jellyfin-opencl-intel
    volumes:
      - ${DOCKERCONFDIR}/jellyfin:/config
      - ${DOCKERSTORAGEDIR}/media:/data/media
    devices:
      - /dev/dri/renderD128:/dev/dri/renderD128
    group_add:
      - "105"   # Numeric render group ID (check with 'getent group render')
      - "44"    # Video group ID for full iGPU access
    restart: unless-stopped
```

---

## Step 5: Service Architecture Explanation

* **`gluetun`**: Acts as your secure VPN gateway tunnel using WireGuard. It establishes a connection (set to NordVPN in Amsterdam) and isolates network traffic.
* **`sabnzbd`**: Usenet client attached directly to Gluetun (`network_mode: "service:gluetun"`). All download activity passes through the encrypted VPN tunnel, while its interface is exposed via port `8080`.
* **`radarr` & `sonarr`**: Movie and TV automation services. Both mount the shared root `/data` path, enabling atomic hardlinks and fast moves across downloads and media folders without copying files.
* **`prowlarr`**: Indexer manager that synchronizes search providers across Radarr and Sonarr automatically.
* **`bazarr`**: Subtitle manager that monitors media folders and downloads matching subtitles.
* **`jellyfin`**: Media server using host networking and passed-through iGPU hardware (`/dev/dri/renderD128`) for video transcoding.

---

## Step 6: Deployment & Network Verification

Download the Docker app from the Ugreen NAS App Center, go to the Project tab, and click the blue Create button. 
Enter a name for the project (in this case, call it arr-stack) and set the storage path to /volume1/docker/arr-stack. 
Note that after entering the project name, the storage path may fill in automatically.
If a warning appears in red stating that a docker-compose.yaml configuration already exists in that storage path, click the import option in/besides that same red text to import the settings. 
After importing, you should see your docker-compose file appear in the editor. 
As long as there are no errors in the imported file, click the blue Deploy button in the lower-right corner to create your Docker containers.

Alternatively, you can start up your Docker containers via the terminal. 
To do this, enable the terminal on your NAS and follow the steps below:
1. Navigate to the compose folder:
   ```bash
   cd /volume1/docker/arr-stack
   ```
2. Launch containers:
   ```bash
   sudo docker compose up -d
   ```
Please note that whenever you make changes to the docker-compose.yaml file, you will need to redeploy the containers for the changes to take effect. 
Simply modifying and saving the file itself (or restarting or stopping the containers) will not automatically update the running containers.

### Check VPN IP Routing
To ensure SABnzbd and Gluetun are properly protected through your VPN:

1. Check your host NAS external IP address:
   ```bash
   curl icanhazip.com
   ```
2. Check the external IP assigned inside the Gluetun container:
   ```bash
   sudo docker exec -it gluetun /bin/sh
   wget -qO - icanhazip.com
   exit
   ```
   *(The IP inside Gluetun must differ from your host NAS IP address).*

3. Access container shell for maintenance if needed:
   ```bash
   sudo docker exec -it sabnzbd /bin/bash
   ```

# Jellyfin Intel QuickSync (QSV) Transcoding Configuration Guide

Use this guide to restore your hardware-accelerated transcoding settings in Jellyfin after a fresh install or reset.

Navigate to **Dashboard** > **Playback** > **Transcoding** and configure the settings as outlined below.

---

## 1. Hardware Acceleration & Decoding
* **Hardware acceleration:** `Intel QuickSync (QSV)`
* **QSV Device:** `/dev/dri/renderD128`

### Enable hardware decoding for:
* [x] H264
* [x] HEVC
* [ ] MPEG2
* [ ] VC1
* [ ] VP8
* [x] VP9
* [x] AV1
* [x] HEVC 10bit
* [x] VP9 10bit
* [ ] HEVC RExt 8/10bit
* [ ] HEVC RExt 12bit
* [ ] Prefer OS native DXVA or VA-API hardware decoders

---

## 2. Hardware Encoding Options
* [x] **Enable hardware encoding**
* [ ] Enable Intel Low-Power H.264 hardware encoder
* [ ] Enable Intel Low-Power HEVC hardware encoder

### Encoding format options:
* [x] **Allow encoding in HEVC format**
* [ ] **Allow encoding in AV1 format** *(Keep unchecked to avoid hardware compatibility crashes)*

---

## 3. Tone Mapping Settings
* [x] **Enable VPP Tone mapping**
  * **VPP Tone mapping brightness gain:** `16`
  * **VPP Tone mapping contrast gain:** `1`
* [x] **Enable Tone mapping**
  * **Select the Tone mapping algorithm to use:** `BT.2390`
  * **Tone mapping mode:** `Auto`
  * **Tone mapping range:** `Auto`
  * **Tone mapping desat:** `0`
  * **Tone mapping peak:** `100`

---

## 4. Advanced & General Settings
* **Transcoding thread count:** `Auto`
* **Max muxing queue size:** `2048`
* **H.265 encoding CRF:** `28`
* **H.264 encoding CRF:** `23`
* **Deinterlacing method:** `Yet Another DeInterlacing Filter (YADIF)`

### Unchecked Options:
* [ ] Double the frame rate when deinterlacing
* [ ] Allow subtitle extraction on the fly
* [ ] Throttle Transcodes
* [ ] Delete segments

---
*Don't forget to click **Save** at the bottom of the page after applying these settings!*
