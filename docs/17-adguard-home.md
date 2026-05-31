# Runbook 17: AdGuard Home

DNS-level ad blocking and privacy protection for every device on the network.

| | |
|---|---|
| **Difficulty** | Beginner |
| **Time Estimate** | 1–2 hours |
| **Runs On** | 2× Raspberry Pi Zero 2 W (OS-level service — not k3s) |
| **Depends On** | Runbook 2 (static IPs assigned on Lab VLAN) |

AdGuard Home runs natively on two dedicated Pi Zero 2 W nodes rather than in the k3s cluster. DNS must stay up independently of cluster reboots and upgrades. Two units give automatic failover: the UDM advertises both IPs as DNS servers via DHCP so if one goes down clients fail over with no manual intervention.

This runbook covers:

1. Installing AdGuard Home on both Pi Zeros
2. Configuring the primary
3. Syncing config to the secondary via adguardhome-sync
4. Advertising both IPs as DNS servers from the UDM

## Prerequisites

- Both Pi Zero 2 Ws provisioned with DietPi (64-bit ARM) and accessible via SSH.
- Static IPs assigned on the Lab VLAN — this runbook uses `10.0.20.20` (primary) and `10.0.20.21` (secondary) as examples. Substitute your actual assignments from Runbook 2.
- UDM admin access.

## Step 1: Install AdGuard Home on Both Pi Zeros

Run the following on **both** nodes. The official automated installer detects ARM64 and downloads the correct binary. See the [official install docs](https://adguard-dns.io/kb/adguard-home/getting-started/) for reference.

```bash
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v
```

The script installs AdGuard Home as a systemd service, starts it, and prints the setup URL.

Verify it is running:

```bash
systemctl status AdGuardHome
```

!!! warning "Port 53 conflict on DietPi"
    DietPi may have `systemd-resolved` listening on port 53. If AdGuard fails to bind, disable it first:
    ```bash
    systemctl disable --now systemd-resolved
    systemctl restart AdGuardHome
    ```

## Step 2: Initial Setup Wizard

Open `http://<pi-ip>:3000` from a browser on your local network and complete the wizard on **both nodes**:

1. **Listen interfaces** — accept the default (all interfaces).
2. **DNS listen port** — leave at 53.
3. **Admin credentials** — create a strong username and password. Save both to Vaultwarden before proceeding. Use the same credentials on both nodes (adguardhome-sync requires it).
4. Complete the wizard. AdGuard Home is now serving DNS on port 53 and its web UI on port 80.

## Step 3: Configure the Primary

Log into the primary at `http://10.0.20.20` and apply the following settings.

**Settings → DNS settings → Upstream DNS servers:**

```
https://dns.quad9.net/dns-query
https://cloudflare-dns.com/dns-query
```

**Bootstrap DNS servers** (used to resolve the upstream DoH hostnames on first start):

```
9.9.9.9
1.1.1.1
```

Enable **Parallel requests** so both upstreams are queried simultaneously.

**Filters → DNS blocklists — enable:**

- AdGuard DNS filter (built-in — enable it)
- Add OISD Basic: `https://abp.oisd.nl/`

The secondary will receive this config automatically in Step 4 — do not configure it manually.

## Step 4: Install adguardhome-sync on the Primary

[adguardhome-sync](https://github.com/bakito/adguardhome-sync) pushes configuration from the primary to the secondary on a schedule. Install it as a Docker container on the **primary Pi Zero**.

First, install Docker on the primary if not already present:

```bash
apt update && apt install -y docker.io
systemctl enable --now docker
```

Create the config directory and write the config file. Create `/etc/adguardhome-sync/adguardhome-sync.yaml` with the following content (substitute your credentials and IPs):

```yaml
origin:
  url: http://localhost:80
  username: admin
  password: YOUR_PRIMARY_PASSWORD

replicas:
  - url: http://10.0.20.21:80
    username: admin
    password: YOUR_SECONDARY_PASSWORD

cron: "*/30 * * * *"
runOnStart: true
```

Run the sync container using the [LinuxServer.io image](https://docs.linuxserver.io/images/docker-adguardhome-sync/) (ARM64 ✅):

```bash
docker run -d \
  --name adguardhome-sync \
  --restart unless-stopped \
  -v /etc/adguardhome-sync:/config \
  lscr.io/linuxserver/adguardhome-sync:latest
```

Verify the first sync completed:

```bash
docker logs adguardhome-sync
```

You should see a successful push. Log into the secondary at `http://10.0.20.21` and confirm blocklists match the primary.

## Step 5: Configure UDM DHCP

In the UniFi Network app, advertise both Pi Zero IPs as DNS servers for each VLAN that should use AdGuard Home:

1. Go to **Settings → Networks → [VLAN name] → DHCP → DNS Server**.
2. Set **DNS Server 1** to `10.0.20.20` (primary).
3. Set **DNS Server 2** to `10.0.20.21` (secondary).
4. Repeat for your Trusted, Lab, and IoT VLANs.

Clients will use AdGuard at the next DHCP renewal. Force a renewal on a test device (`sudo dhclient -r && sudo dhclient` on Linux, reconnect Wi-Fi on a phone) and verify queries appear in AdGuard's **Query Log**.

## Ansible

The Pi Zero 2 W nodes should be in your `homelab-ansible` inventory so their configuration is rebuild-durable. Add them to a `dns` host group with a role that covers:

- AdGuard Home install and service enable
- systemd-resolved disable
- Docker install and adguardhome-sync container

## Verification

- [ ] AdGuard Home service running on both nodes:

    ```bash
    systemctl status AdGuardHome
    ```

- [ ] Primary web UI accessible at `http://10.0.20.20`.
- [ ] Secondary web UI accessible at `http://10.0.20.21` — blocklists and filter rules match primary.
- [ ] adguardhome-sync container running on primary with no errors in logs.
- [ ] Test that a known ad domain is blocked from a network client:

    ```bash
    dig doubleclick.net @10.0.20.20
    # Expect: 0.0.0.0 in the answer section
    ```

- [ ] UDM DHCP advertising both Pi Zero IPs as DNS on all VLANs.
- [ ] Query log in AdGuard UI shows traffic from network devices.
