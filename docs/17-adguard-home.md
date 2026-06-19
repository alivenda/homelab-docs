# Runbook 17: AdGuard Home

DNS-level ad blocking and privacy protection for every device on the network.

| | |
|---|---|
| **Difficulty** | Beginner |
| **Time Estimate** | 1 hour |
| **Runs On** | 1× Raspberry Pi (OS-level service — not k3s). This build: a Pi 3 Model B (`pyrite`). |
| **Depends On** | Runbook 2 (static IP on the Default VLAN) |

AdGuard Home runs natively on a dedicated Raspberry Pi rather than in the k3s cluster: DNS must stay up independently of cluster reboots and upgrades. **One Pi is enough** for a working setup. Adding a **second** Pi is an optional upgrade for automatic failover — the UDM advertises both IPs as DNS servers via DHCP, so if one goes down clients fail over with no manual intervention.

This build runs a single node, `pyrite`, at `10.0.0.20` on the Default VLAN. The optional secondary is `marcasite` at `10.0.0.21`.

!!! note "Why the Default VLAN, not Lab"
    DNS is shared infrastructure, not a cluster service, so it lives on the Default LAN (the management plane) alongside the UDM — not on Lab, where the firewall posture is "initiates to nothing." Trusted devices already reach the Default LAN (the `trusted-to-internal-allow` policy from R2), so they get DNS with no extra rule; IoT and Lab need one small allow rule each — see [R2 Step 3d](02-networking.md#step-3d-dns-enforcement).

This runbook covers:

1. Installing AdGuard Home on the Pi
2. Configuring upstreams and blocklists
3. Advertising it as the DNS server from the UDM
4. *(Optional)* Adding a second Pi for failover with config sync

## Prerequisites

- A Raspberry Pi provisioned with DietPi (64-bit ARM) and reachable via SSH. If your `dietpi.txt` included `AUTO_SETUP_INSTALL_SOFTWARE_ID=126` (see [Runbook 3](03-turing-pi.md) for the DietPi automation pattern), AdGuard Home is already installed — skip to Step 2.
- Static IP `10.0.0.20` on the Default VLAN (set via `dietpi.txt` at first boot, per Runbook 2).
- UDM admin access.

## Step 1: Install AdGuard Home

If DietPi-Software didn't already install it (ID 126), run the official installer — it detects ARM64 and downloads the correct binary. See the [official install docs](https://adguard-dns.io/kb/adguard-home/getting-started/) for reference.

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

Open `http://10.0.0.20:3000` from a browser on your local network and complete the wizard:

1. **Listen interfaces** — accept the default (all interfaces).
2. **DNS listen port** — leave at 53.
3. **Admin credentials** — create a strong username and password; save both to Vaultwarden. (If you plan to add a second node later, reuse these credentials there — adguardhome-sync requires matching logins.)
4. Complete the wizard. AdGuard Home now serves DNS on port 53 and its web UI on port 80.

## Step 3: Configure Upstreams and Blocklists

Log into `http://10.0.0.20` and apply the following.

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

## Step 4: Point the UDM's DHCP at AdGuard

In the UniFi Network app, hand out the Pi's IP as the DNS server for each VLAN that should use AdGuard Home:

1. Go to **Settings → Networks → [VLAN name] → DHCP → DNS Server**.
2. Set **DNS Server 1** to `10.0.0.20`.
3. Repeat for the Trusted, Lab, and IoT VLANs.

!!! warning "IoT and Lab need a firewall rule to reach it"
    AdGuard sits on the Default LAN (Internal zone). Trusted reaches it already, but with the zone-based firewall the IoT and Lab zones are blocked from Internal by default — clients there will *silently* lose DNS unless you add the allow rules in [R2 Step 3d](02-networking.md#step-3d-dns-enforcement). Add those before flipping each VLAN's DNS over.

Clients pick up AdGuard at the next DHCP renewal. Force a renewal on a test device (`sudo dhclient -r && sudo dhclient` on Linux, reconnect Wi-Fi on a phone) and verify queries appear in AdGuard's **Query Log**.

## Optional: Add a Second Node for Failover

A single Pi is a single point of failure for DNS. For automatic failover, add a second Pi (`marcasite`, `10.0.0.21`) and have the UDM hand out both. Config is kept in sync from the primary, so you only ever edit one.

### Install and set up the secondary

Repeat Step 1 and Step 2 on the second Pi at `10.0.0.21`, using the **same** admin username and password as the primary. Do **not** configure upstreams or blocklists on it — they'll be pushed from the primary in the next step.

### Sync config with adguardhome-sync

[adguardhome-sync](https://github.com/bakito/adguardhome-sync) pushes configuration from the primary to the secondary on a schedule. Run it as a Docker container on the **primary** (`pyrite`).

Install Docker on the primary if not already present:

```bash
apt update && apt install -y docker.io
systemctl enable --now docker
```

Create `/etc/adguardhome-sync/adguardhome-sync.yaml` (substitute your credentials):

```yaml
origin:
  url: http://localhost:80
  username: admin
  password: YOUR_PRIMARY_PASSWORD

replicas:
  - url: http://10.0.0.21:80
    username: admin
    password: YOUR_SECONDARY_PASSWORD

cron: "*/30 * * * *"
runOnStart: true
```

Run the sync container ([LinuxServer.io image](https://docs.linuxserver.io/images/docker-adguardhome-sync/), ARM64 ✅):

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

You should see a successful push. Log into `http://10.0.0.21` and confirm blocklists match the primary.

### Advertise both from the UDM

Back in **Settings → Networks → [VLAN name] → DHCP → DNS Server**, set **DNS Server 2** to `10.0.0.21` on each VLAN (DNS Server 1 stays `10.0.0.20`). Clients now fail over automatically. Extend the IoT/Lab allow rules in [R2 Step 3d](02-networking.md#step-3d-dns-enforcement) to cover `10.0.0.21` as well.

## Ansible

The Pi should be in your `homelab-ansible` inventory so its configuration is rebuild-durable. Add it to a `dns` host group with a role that covers:

- AdGuard Home install and service enable
- systemd-resolved disable
- *(secondary only)* Docker install and adguardhome-sync container

## Verification

- [ ] AdGuard Home service running:

    ```bash
    systemctl status AdGuardHome
    ```

- [ ] Web UI accessible at `http://10.0.0.20`.
- [ ] A known ad domain is blocked:

    ```bash
    dig doubleclick.net @10.0.0.20
    # Expect: 0.0.0.0 in the answer section
    ```

- [ ] UDM DHCP advertising `10.0.0.20` as DNS on every VLAN that should filter.
- [ ] From a device on Trusted, IoT, and Lab: DNS resolves (proves the R2 Step 3d allow rules are in place).
- [ ] Query log in AdGuard UI shows traffic from network devices.

**If you added the second node:**

- [ ] Secondary web UI at `http://10.0.0.21` — blocklists and filter rules match the primary.
- [ ] adguardhome-sync container running on the primary with no errors in logs.
- [ ] UDM DHCP advertising both `10.0.0.20` and `10.0.0.21`.
