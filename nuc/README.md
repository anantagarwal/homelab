# Intel NUC (Proxmox) Setup Guide

Target host: Intel Core i3-8109U (32 GB RAM, 1 TB ZFS pool `zfspool`).
`media` CT static IP: `192.168.1.202`.

---

## 1. Create ZFS Datasets

> **Where to run:** **Proxmox VE Host** (Proxmox Web UI → Node → **Shell**, or SSH to the Proxmox host — **not** inside a CT).

Create dedicated datasets on `zfspool` so storage is cleanly isolated and quota-limited:

```bash
# Media dataset (100 GB quota) — shared between share CT and media CT
zfs create -o quota=100G zfspool/media
mkdir -p /zfspool/media/{movies,shows,music}

# Paperless dataset (20 GB quota) — consume/ is also shared with share CT
zfs create -o quota=20G zfspool/paperless
mkdir -p /zfspool/paperless/{data,media,consume,export,redis}

# Allow unprivileged LXCs (UID/GID 100000+ mapping) to read/write cleanly
chmod -R 777 /zfspool/media /zfspool/paperless/consume
```

> **Tip:** `chmod -R 777` on the shared drop folders (`/zfspool/media` and `/zfspool/paperless/consume`) is the simplest way to let both an unprivileged `share` CT and an unprivileged `media` CT read/write without custom UID shift maps. For tighter permissions later, map a shared GID across both CTs.

---

## 2. Create & Configure the `media` CT

> **Where to run:** **Proxmox VE Host** (same Proxmox host shell as Step 1).

Recommended CT specs for CTID `202` (matching `192.168.1.202`, in series with `share` CT `201`):

| Setting | Value | Why |
|---|---|---|
| **Template** | Ubuntu 24.04 LTS or Debian 12 | Lightweight, official Docker support |
| **Type** | Unprivileged | Proxmox default & safer |
| **Cores** | `4` | Lets Jellyfin/Paperless burst across all 4 threads |
| **Memory** | `4096` MiB (4 GB) | Plenty for Jellyfin + Paperless + Tika + Gotenberg |
| **Swap** | `2048` MiB (2 GB) | Safety buffer |
| **Root Disk** | `32` GB on Proxmox default storage (e.g. `local-lvm`) | OS + Docker images & layers (keep `zfspool` only for shared data) |
| **Features** | `nesting=1,keyctl=1` | Required for Docker inside unprivileged LXC |
| **Network** | `vmbr0`, `192.168.1.202/24`, GW `192.168.1.1` | Static IP for Traefik forwarding |

### Bind-mount ZFS datasets into both CTs (run on Proxmox VE Host)

```bash
# media CT (CTID 202)
pct set 202 -mp0 /zfspool/media,mp=/mnt/media
pct set 202 -mp1 /zfspool/paperless,mp=/mnt/paperless

# Existing share CT (CTID 201 — keeps your existing 200G mp0:/data untouched)
pct set 201 -mp1 /zfspool/media,mp=/mnt/media
pct set 201 -mp2 /zfspool/paperless/consume,mp=/mnt/paperless-consume
```

### Enable Intel QuickSync (`/dev/dri`) for Jellyfin (run on Proxmox VE Host)

Append the following to `/etc/pve/lxc/202.conf` on the Proxmox host (verify `/dev/dri` devices first with `ls -la /dev/dri`):

```ini
lxc.cgroup2.devices.allow: c 226:* rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
```

Then restart the CT: `pct reboot 202`.

---

## 3. Expose `media` & `paperless-consume` in `share` CT

> **Where to run:** **Inside `share` CT** (`ssh share` or `pct enter 201` from the Proxmox host).

Add these shares to `/etc/samba/smb.conf` (matching your existing `[data]` share's `anant` user/masks) and restart `smbd` (`systemctl restart smbd`):

```ini
[media]
   path = /mnt/media
   force user = anant
   force group = anant
   valid users = anant
   create mask = 0774
   force create mode = 0774
   directory mask = 0775
   force directory mode = 0775
   browseable = yes
   writable = yes
   read only = no
   guest ok = no

[paperless-consume]
   path = /mnt/paperless-consume
   force user = anant
   force group = anant
   valid users = anant
   create mask = 0774
   force create mode = 0774
   directory mask = 0775
   force directory mode = 0775
   browseable = yes
   writable = yes
   read only = no
   guest ok = no
```

Now from Windows you can drop movies/shows into `\\<SAMBA_IP>\media` and PDFs/scans into `\\<SAMBA_IP>\paperless-consume` using your existing `anant` Samba credentials.

---

## 4. Create `anant` User, Install Docker & Start the Stack

> **Where to run:** **Inside `media` CT** (`pct enter 202` from the Proxmox host, or SSH to `192.168.1.202`).

As `root` inside CT `202`, install prerequisites, create the `anant` user with `sudo` and `docker` group access, and switch to `anant`:

```bash
apt update && apt install -y curl git ca-certificates sudo openssh-server
systemctl enable --now ssh
curl -fsSL https://get.docker.com | sh

# Create user `anant`, set password, and grant sudo + docker access
useradd -m -s /bin/bash anant
passwd anant
usermod -aG sudo,docker anant

# Switch to `anant` for all remaining steps (or SSH in as anant@192.168.1.202)
su - anant
```

Now as `anant`:

```bash
git clone <your-repo-url> ~/homelab
cd ~/homelab
chmod +x hl

# Create external Docker network
docker network create homelab-network

# Configure Paperless secrets
cp nuc/paperless-ngx/.env.example nuc/paperless-ngx/.env
nano nuc/paperless-ngx/.env

# Start Jellyfin + Paperless-NGX
./hl nuc up
```

---

## 5. Reload Traefik & Glance on the Raspberry Pi

> **Where to run:** **Raspberry Pi** (`ssh pi` / your Pi host).

Pull the updated repo on the Raspberry Pi and run `./hl restart` (no service name) so Traefik + Glance pick up the new routes and the old `paperless*` / `docling` orphan containers on the Pi are stopped and removed:

```bash
cd ~/homelab
git pull
./hl restart
```

*(If you want to keep the old Pi containers running temporarily until the NUC is verified, run `./hl restart traefik glance` instead, then run `./hl restart` later to clean them up.)*

Both `https://jellyfin.internal.anantagarwal.me` and `https://paperless.internal.anantagarwal.me` will now be served with Cloudflare TLS via your Pi's Traefik.
