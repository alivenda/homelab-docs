# Runbook 32: Syncthing

Continuous peer-to-peer file synchronisation between devices — no cloud relay needed.

| | |
|---|---|
| **Difficulty** | Beginner |
| **Time Estimate** | 30 minutes per device |
| **Runs On** | Per-device install (not k3s) |
| **Depends On** | Runbook 2 (static IPs on your LAN) |

Syncthing ([syncthing.net](https://syncthing.net)) synchronises folders between devices directly — no central server is required. Each device runs its own Syncthing instance; devices pair with each other using device IDs, and sync happens over your LAN or via Syncthing's global relay network when devices are remote. ARM64 ✅ (native binary and Docker image available). See the [Syncthing documentation](https://docs.syncthing.net) for full reference.

**Deployment model:** Syncthing is **not** deployed on k3s. Each device that participates in sync runs its own instance. This runbook covers: your main machine, the NAS, and any other devices you want to include.

!!! note "Why not on k3s?"
    Syncthing in a cluster pod adds latency and complexity without benefit — pods are ephemeral and don't have stable identities. Run Syncthing natively on the machines whose filesystems you want to sync.

## Step 1: Install Syncthing on Each Device

### Your main machine (CachyOS)

Install from the Arch/CachyOS repository:

```bash
sudo pacman -S syncthing
```

Enable and start the user service:

```bash
systemctl --user enable --now syncthing
```

The web UI is available at `http://localhost:8384`.

### NAS (UGREEN DXP6800 Pro)

Run Syncthing as a Docker container on the NAS:

```yaml
services:
  syncthing:
    image: lscr.io/linuxserver/syncthing:latest
    container_name: syncthing
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/London
    volumes:
      - /path/to/syncthing/config:/config
      - /volume1/sync:/sync     # mount whatever NAS folders you want to sync
    ports:
      - 8384:8384    # Web UI
      - 22000:22000  # Sync protocol TCP
      - 22000:22000/udp
      - 21027:21027/udp  # Discovery broadcast
    restart: unless-stopped
```

Deploy via the NAS Docker interface or `docker compose up -d`.

### Other devices (Linux servers, Raspberry Pi)

Install via the official script or package manager, then enable the service:

```bash
# Debian/Ubuntu/DietPi
curl -s https://syncthing.net/release-key.txt | gpg --dearmor > /usr/share/keyrings/syncthing-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/syncthing-archive-keyring.gpg] https://apt.syncthing.net/ syncthing stable" > /etc/apt/sources.list.d/syncthing.list
apt update && apt install syncthing
systemctl enable --now syncthing@<username>
```

## Step 2: Pair Devices

1. Open each device's Syncthing web UI at `http://<device-ip>:8384`.
2. On device A, go to **Remote Devices → Add Remote Device**.
3. Enter device B's **Device ID** (shown under **Actions → Show ID** on device B).
4. Accept the pairing request on device B.
5. Repeat until all devices are paired.

## Step 3: Share Folders

1. On the device that holds the source files, go to **Folders → Add Folder**.
2. Set the **Folder Path** to the local directory you want to sync.
3. In the **Sharing** tab, tick the remote devices that should receive this folder.
4. Accept the folder share on each remote device and specify a local path on that device.

**Suggested folders to sync:**

| Folder | Source | Destination |
|--------|--------|-------------|
| Documents | Main machine | NAS |
| Scripts / dotfiles | Main machine | NAS (backup) |
| Audiobooks | NAS | Main machine (optional) |

## Step 4: Secure the Web UI

By default the Syncthing web UI is unauthenticated and bound to `localhost`. For the NAS instance, since the UI is exposed on a LAN port:

1. Open the NAS Syncthing UI at `http://<nas-ip>:8384`.
2. Go to **Actions → Settings → GUI**.
3. Set a **GUI Authentication User** and **Password**. Save these to Vaultwarden.
4. Optionally restrict the **GUI Listen Address** to `127.0.0.1:8384` if you don't need remote UI access.

## Verification

- [ ] Syncthing running on your main machine:

    ```bash
    systemctl --user status syncthing
    ```

- [ ] Syncthing running on NAS (check container logs).
- [ ] All devices show as **Connected** in each other's web UI.
- [ ] A test file placed in a shared folder on one device appears on the other within seconds.
- [ ] NAS Syncthing UI password saved to Vaultwarden.
