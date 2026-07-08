# AdGuard Home

!!! success "Status — Live"
    Live on a dedicated Raspberry Pi (`pyrite`, `10.0.0.20`) — not k3s.

DNS-level ad blocking and privacy protection for every device on the network.

| | |
|---|---|
| **Primary** | `pyrite` — `10.0.0.20`, web UI `:8083` |
| **Secondary (optional)** | `marcasite` — `10.0.0.21`, web UI `:8083` |
| **OS** | DietPi (64-bit ARM) |
| **Runs On** | 1× Raspberry Pi (OS-level service — not k3s). This build: a Pi 3 Model B (`pyrite`). |
| **Depends On** | Networking (static IP on the Default VLAN) |
| **Difficulty** | Beginner |
| **Time Estimate** | 1 hour |

AdGuard Home runs natively on a dedicated Raspberry Pi rather than in the k3s cluster: DNS must stay up independently of cluster reboots and upgrades. **One Pi is enough** for a working setup. Adding a **second** Pi is an optional upgrade for automatic failover — the UDM advertises both IPs as DNS servers via DHCP, so if one goes down clients fail over with no manual intervention.

This build runs a single node, `pyrite`, at `10.0.0.20` on the Default VLAN. The optional secondary is `marcasite` at `10.0.0.21`.

!!! note "Why the Default VLAN, not Lab"
    DNS is shared infrastructure, not a cluster service, so it lives on the Default LAN (the management plane) alongside the UDM — not on Lab, where the firewall posture is "initiates to nothing." Trusted devices already reach the Default LAN (the `trusted-to-internal-allow` policy from Networking), so they get DNS with no extra rule; IoT and Lab need one small allow rule each — see [Networking Step 3d](networking.md#step-3d-dns-enforcement).

This runbook covers:

1. Installing AdGuard Home on the Pi
2. Configuring upstreams and blocklists
3. Advertising it as the DNS server from the UDM
4. *(Optional)* Adding a second Pi for failover with config sync

## Prerequisites

- A Raspberry Pi provisioned with DietPi (64-bit ARM) and reachable via SSH. If your `dietpi.txt` included `AUTO_SETUP_INSTALL_SOFTWARE_ID=126` (see [Turing Pi](turing-pi.md) for the DietPi automation pattern), AdGuard Home is already installed *and* its web-admin login was seeded from `AUTO_SETUP_GLOBAL_PASSWORD` — there's no wizard to run. Rotate that password in [Post-install hardening](#post-install-hardening) first, then continue at Step 3.
- Static IP `10.0.0.20` on the Default VLAN (set via `dietpi.txt` at first boot, per Networking).
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

!!! note "Headless installs skip this"
    If AdGuard came in via DietPi-Software (ID 126), there's no wizard — it was pre-configured and its admin login seeded from `AUTO_SETUP_GLOBAL_PASSWORD`. Rotate that password in [Post-install hardening](#post-install-hardening), then go to Step 3.

Open `http://10.0.0.20:3000` from a browser on your local network and complete the wizard:

1. **Listen interfaces** — accept the default (all interfaces).
2. **DNS listen port** — leave at 53.
3. **Admin credentials** — create a strong username and password; save both to Vaultwarden. (If you plan to add a second node later, reuse these credentials there — adguardhome-sync requires matching logins.)
4. Complete the wizard. AdGuard Home now serves DNS on port 53 and its web UI on port 80 (a manual install's default).

!!! info "Web-UI port: `:8083` on this build"
    DietPi-Software (ID 126) serves the web UI on **`:8083`**, *not* `:80` — and never on `:3000`, which is only the manual-install setup wizard. This runbook writes `http://10.0.0.20:8083` for `pyrite`; on a manual install, drop the `:8083` (it's on `:80`). The host firewall in the [`dns` Ansible play](#ansible) opens whichever port `adguard_web_port` is set to.

## Step 3: Configure Upstreams and Blocklists

Log into `http://10.0.0.20:8083` and apply the following.

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

## Recommended configuration

Step 3 gets DNS working; these settings make it noticeably better on a homelab. All live in the web UI, so the nightly backup captures them.

### Split-horizon DNS for local services

By default a client resolving `vault.yourdomain.com` goes out to public DNS and hairpins back in through the WAN. Point internal lookups straight at the reverse proxy instead.

**Filters → DNS rewrites → Add:**

| Domain | Answer |
|---|---|
| `*.yourdomain.com` | `10.0.20.200` (Traefik's load-balancer IP) |

Internal clients now reach every gateway-routed service directly. TLS still validates — Traefik selects the certificate by SNI — and the public DNS records stay in place for off-network access. This is the AdGuard-native version of the UDM host-record trick in [Traefik](traefik.md), and it's the one that applies now: every VLAN resolves through AdGuard, not the UDM.

!!! warning "The wildcard assumes everything is behind the proxy"
    It sends *every* `*.yourdomain.com` name to Traefik. If a subdomain is served elsewhere, add a more specific rewrite for it — AdGuard honours the most specific match — or use per-host rewrites instead of the wildcard.

### DNSSEC

In **Settings → DNS settings**, tick **Enable DNSSEC** to validate signed responses from the upstreams.

### Private reverse DNS

So the query log shows device names instead of bare IPs: under **Settings → DNS settings → Private reverse DNS servers** add your gateway (`10.0.0.1`), then enable **Use private reverse DNS resolvers** and **reverse-resolve clients' IP addresses**.

### Cache tuning

Under **Settings → DNS settings → DNS cache configuration**, set the cache size to `4194304` (4 MB) and enable **optimistic caching** — repeat lookups return instantly from cache while AdGuard refreshes them in the background.

### Query log retention

The Pi logs to its SD card, so cap retention in **Settings → General settings**: a **7-day** query log is plenty for debugging. Statistics are tiny and can stay longer.

## Step 4: Point the UDM's DHCP at AdGuard

In the UniFi Network app, hand out the Pi's IP as the DNS server for each VLAN that should use AdGuard Home:

1. Go to **Settings → Networks → [VLAN name] → DHCP → DNS Server**.
2. Set **DNS Server 1** to `10.0.0.20`.
3. Repeat for the Trusted, Lab, and IoT VLANs.

!!! warning "IoT and Lab need a firewall rule to reach it"
    AdGuard sits on the Default LAN (Internal zone). Trusted reaches it already, but with the zone-based firewall the IoT and Lab zones are blocked from Internal by default — clients there will *silently* lose DNS unless you add the allow rules in [Networking Step 3d](networking.md#step-3d-dns-enforcement). Add those before flipping each VLAN's DNS over.

Clients pick up AdGuard at the next DHCP renewal. Force a renewal on a test device (`sudo dhclient -r && sudo dhclient` on Linux, reconnect Wi-Fi on a phone) and verify queries appear in AdGuard's **Query Log**.

## Post-install hardening { #post-install-hardening }

A fresh DietPi + AdGuard install leaves a couple of doors wider than they need to be. Close them before the box starts resolving DNS for the whole network.

### Give AdGuard its own admin password

A headless install (DietPi-Software ID 126) seeds the web-admin login from `AUTO_SETUP_GLOBAL_PASSWORD` — the **same** secret as the `root` and `dietpi` OS accounts. One leaked password would then surrender both the box and its DNS config, so decouple them.

AdGuard Home has no "change password" button in the web UI; the admin hash lives in its config file. Generate a fresh bcrypt hash (`htpasswd` ships in `apache2-utils`):

```bash
apt install -y apache2-utils
htpasswd -B -C 10 -n admin
# Enter a new, unique password when prompted.
# Output: admin:$2y$10$....
```

Copy the hash (everything after `admin:`) into the `users:` block of `AdGuardHome.yaml` (on this build: `/mnt/dietpi_userdata/adguardhome/AdGuardHome.yaml`):

```yaml
users:
  - name: admin
    password: $2y$10$....your.new.hash....
```

Restart AdGuard, then log in at `http://10.0.0.20:8083` with the new password to confirm:

```bash
dietpi-services restart adguardhome
```

Save the new password to Vaultwarden, separate from the OS credentials.

### Rotate the OS logins and lock SSH to keys

While you're in here, rotate `root` and `dietpi` off the shared global password too (`passwd root`, `passwd dietpi`), and switch SSH to key-only. Add your public key first (`ssh-copy-id dietpi@10.0.0.20`) and confirm a passwordless login works, **then** disable password auth with DietPi's helper:

```bash
sudo /boot/dietpi/func/dietpi-set_software disable_ssh_password_logins 1
```

!!! warning "The helper leaves a PAM gap"
    `disable_ssh_password_logins` writes `PasswordAuthentication no` but not `KbdInteractiveAuthentication no`, so OpenSSH's PAM keyboard-interactive path can still prompt for a password. Verify both are closed:
    ```bash
    sudo sshd -T | grep -E 'passwordauthentication|kbdinteractive'
    # both should report `no`
    ```
    If `kbdinteractiveauthentication` is still `yes`, drop `KbdInteractiveAuthentication no` into a separate `/etc/ssh/sshd_config.d/99-hardening.conf` (DietPi won't clobber it) and restart ssh.

!!! tip "This is codified"
    All of the above lives in `homelab-ansible`: the `dns` play installs AdGuard and applies a DNS-shaped firewall, and the shared SSH play sets key-only auth **and** closes the `KbdInteractiveAuthentication` gap on every node. Re-running the playbook keeps SSH hardened — but the admin-password rotation stays a one-time manual step, since the automation never generates the secret.

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
  url: http://localhost:8083
  username: admin
  password: YOUR_PRIMARY_PASSWORD

replicas:
  - url: http://10.0.0.21:8083
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

You should see a successful push. Log into `http://10.0.0.21:8083` and confirm blocklists match the primary.

### Advertise both from the UDM

Back in **Settings → Networks → [VLAN name] → DHCP → DNS Server**, set **DNS Server 2** to `10.0.0.21` on each VLAN (DNS Server 1 stays `10.0.0.20`). Clients now fail over automatically. Extend the IoT/Lab allow rules in [Networking Step 3d](networking.md#step-3d-dns-enforcement) to cover `10.0.0.21` as well.

## Ansible

`pyrite` is in the `homelab-ansible` inventory under a `dns` host group, so its configuration is rebuild-durable. The `dns` play (tag `dns`) covers:

- AdGuard Home install (idempotent — gated on the running unit) and service enable
- a DNS-shaped UFW ruleset (port 53 from every VLAN; SSH + web UI from Trusted only)
- a nightly `AdGuardHome.yaml` → Garage (S3) backup timer

It also inherits the shared `all` hardening plays (key-only SSH, unattended-upgrades, chrony). Apply with:

```bash
ansible-playbook site.yml --limit pyrite
```

*(secondary only)* Docker and the adguardhome-sync container aren't codified yet — add them to the play when you bring up `marcasite`.

## Verification

- [ ] AdGuard Home service running:

    ```bash
    systemctl status AdGuardHome
    ```

- [ ] Web UI accessible at `http://10.0.0.20:8083`.
- [ ] A known ad domain is blocked:

    ```bash
    dig doubleclick.net @10.0.0.20
    # Expect: 0.0.0.0 in the answer section
    ```

- [ ] UDM DHCP advertising `10.0.0.20` as DNS on every VLAN that should filter.
- [ ] From a device on Trusted, IoT, and Lab: DNS resolves (proves the Networking Step 3d allow rules are in place).
- [ ] Query log in AdGuard UI shows traffic from network devices.

**If you added the second node:**

- [ ] Secondary web UI at `http://10.0.0.21:8083` — blocklists and filter rules match the primary.
- [ ] adguardhome-sync container running on the primary with no errors in logs.
- [ ] UDM DHCP advertising both `10.0.0.20` and `10.0.0.21`.
