# Runbook 2: Network Segmentation and VPN

VLAN plan, firewall, and remote access via Tailscale. Sets up the network model that every later runbook assumes (cluster nodes on Lab VLAN, MetalLB pool reserved, mDNS scoped, Tailscale subnet routing).

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Time Estimate** | 2–3 hours |
| **Runs On** | UDM + a Turing Pi node for Tailscale subnet routing |
| **Prerequisites** | UDM powered and reachable on default LAN |
| **Targets** | UniFi OS 5.x · Network application 10.x |

!!! warning "UDM version assumption"
    Step 3 (firewall) targets the **Policy Engine / Zone-Based Firewall** introduced in UniFi OS 4 and current in 5.x. On UniFi OS 3.x or earlier, the firewall is configured under **Settings → Firewall → LAN IN** with a different model (default-allow + targeted drops) and these steps do not apply 1:1. Check **System → About** in the UniFi UI to confirm your version.

## Network Plan

This table is the authoritative source for every later runbook (Terraform `unifi_network` resources, Ansible inventory IPs, MetalLB pool config). Change it here first, propagate everywhere else.

| VLAN | Name | ID | Subnet | Gateway | Static range | DHCP range | mDNS |
|---|---|---|---|---|---|---|---|
| (untagged) | default | — | `10.0.0.0/24` | `10.0.0.1` | `.2–.99` | `.100–.250` | off |
| 10 | trusted | 10 | `10.0.10.0/24` | `10.0.10.1` | `.10–.99` | `.100–.250` | **on** |
| 20 | lab | 20 | `10.0.20.0/24` | `10.0.20.1` | `.10–.99` | `.100–.199` | off |
| 30 | iot | 30 | `10.0.30.0/24` | `10.0.30.1` | `.10–.99` | `.100–.250` | **on** |

**Globals**

- DHCP lease: 86400s (1 day)
- IPv6: disabled on both WAN and LAN (see Step 1 note)
- IGMP snooping: off
- UDM upstream DNS: `1.1.1.1` (Cloudflare) — migrate to in-cluster AdGuard later
- MetalLB pool: `10.0.20.200–.250` (reserved within Lab VLAN, outside DHCP)

### Static IP allocations

Within the static range of each VLAN. Cluster nodes are assigned via netplan on the node (not UDM DHCP reservation) so they boot deterministically even if UDM is down.

| Device | VLAN | IP | Assignment |
|---|---|---|---|
| UDM | default | `10.0.0.1` | UDM default |
| UniFi switch | default | `10.0.0.2` | UDM DHCP reservation |
| UniFi AP | default | `10.0.0.3` | UDM DHCP reservation |
| Turing Pi 2 BMC | default | `10.0.0.4` | UDM DHCP reservation |
| ruby (k3s control plane, Tailscale subnet router) | lab | `10.0.20.10` | netplan on node |
| emerald (Tailscale subnet router failover) | lab | `10.0.20.11` | netplan on node |
| topaz | lab | `10.0.20.12` | netplan on node |
| amethyst | lab | `10.0.20.13` | netplan on node |
| UGREEN DXP6800 Pro NAS | lab | `10.0.20.50` | UDM DHCP reservation |
| Apple TV | iot | `10.0.30.10` | UDM DHCP reservation |

### WiFi SSID mapping

| SSID | VLAN | Devices |
|---|---|---|
| `home` | 10 (trusted) | phones, laptops |
| `home-iot` | 30 (iot) | Apple TV, smart home devices |

Lab VLAN is wired only — no SSID.

## Topology

```text
Internet
   │
   ▼
┌──────────────────────────────────────────────────────────────┐
│ UDM (10.0.0.1)                                               │
│   ├─ Default LAN  10.0.0.0/24    gateway: 10.0.0.1           │
│   ├─ VLAN 10  Trusted  10.0.10.0/24   gateway: 10.0.10.1     │
│   ├─ VLAN 20  Lab      10.0.20.0/24   gateway: 10.0.20.1     │
│   └─ VLAN 30  IoT      10.0.30.0/24   gateway: 10.0.30.1     │
└────────────────────────┬─────────────────────────────────────┘
                         │
                ┌────────┴────────┐
                │                 │
        ┌───────▼──────┐   ┌──────▼──────┐
        │ UniFi Switch │   │  UniFi AP   │
        └──────────────┘   └─────────────┘

   Default LAN  10.0.0.0/24                  [mDNS off]
     └─ UDM, switch, AP management

   VLAN 10  Trusted  10.0.10.0/24            [mDNS on]
     ├─ SSID "home"
     └─ phones · laptops · home desktop

   VLAN 20  Lab  10.0.20.0/24                [mDNS off]
     ├─ ruby      .10   k3s control plane + Tailscale subnet router
     ├─ emerald   .11   k3s worker + Tailscale subnet router (failover)
     ├─ topaz     .12   k3s worker
     ├─ amethyst  .13   k3s worker
     ├─ NAS       .50   UGREEN DXP6800 Pro (NFS PVs, MinIO S3, Plex, Immich)
     └─ MetalLB pool  .200–.250  (Traefik VIP and other LoadBalancer services)

   VLAN 30  IoT  10.0.30.0/24                [mDNS on]
     ├─ SSID "home-iot"
     └─ Apple TV .10 · smart home devices

   mDNS reflector:  Trusted ↔ IoT only  (for AirPlay/Bonjour discovery)

   Tailscale subnet routes advertised by ruby:
     10.0.0.0/24    (Default — UDM admin from afar)
     10.0.10.0/24   (Trusted — reach home desktop remotely)
     10.0.20.0/24   (Lab — cluster admin)
     IoT (10.0.30.0/24) intentionally NOT advertised
```

## VLAN Architecture

Each VLAN has a distinct trust level and a deliberate reason for existing:

- **Default LAN — `10.0.0.0/24`:** infrastructure management plane. UDM, switch, AP all live here. Treat as semi-trusted; you do not want random devices landing on it.
- **VLAN 10 — Trusted (`10.0.10.0/24`):** your personal devices. Phones, laptops, home desktop. mDNS is on so casting/AirPlay/Bonjour to the IoT VLAN works without you switching networks.
- **VLAN 20 — Lab (`10.0.20.0/24`):** the k3s cluster. Wired only, no SSID, mDNS off. Everything here is yours but the blast radius if a workload escapes the cluster is contained to this VLAN.
- **VLAN 30 — IoT (`10.0.30.0/24`):** smart home gear including the Apple TV. mDNS is on so phones on Trusted can discover the Apple TV for AirPlay. Internet egress allowed (most IoT needs cloud services) but no inbound from anywhere except specific Home Assistant flows.

!!! note "Why Apple TV is on IoT and not Trusted"
    Apple TV is a closed-firmware appliance that talks to Apple's cloud services. Putting it on Trusted would let it reach your personal devices unnecessarily. Putting it on IoT with mDNS reflector enabled is the Ubiquiti-recommended pattern: segmentation preserved, AirPlay still works because mDNS announcements flow between Trusted and IoT.

## Step 1: Create VLANs on UDM

In UniFi Network → **Settings → Networks**, create one network per VLAN row in the table above. For each:

1. **Name** and **VLAN ID** as in the Network Plan.
2. **Gateway/Subnet:** matches the table.
3. **DHCP Mode:** DHCP Server. Set **DHCP Range** to the values from the table (Lab uses `.100–.199`, others use `.100–.250`).
4. **DHCP DNS Server:** Auto (UDM). UDM forwards to upstream — set the upstream in Step 1a below.
5. **Multicast DNS:** **on** for Trusted and IoT, **off** for Lab and Default.
6. **IGMP Snooping:** off.
7. **IPv6 Interface Type:** None (disable).

### Step 1a: UDM upstream DNS

UDM Settings → **Internet → Primary Connection → Advanced → DNS Server**: set to `1.1.1.1` and `1.0.0.1` (or your preferred upstream — Quad9 `9.9.9.9` is the other common choice).

### Step 1b: Reserve the MetalLB range from DHCP

In UniFi → Networks → **Lab** → DHCP → **Advanced → DHCP Exclude IP Range**, add `10.0.20.200–10.0.20.250`. The DHCP pool is already limited to `.100–.199`, but the exclude range belt-and-suspenders prevents an accidental DHCP-range widening from colliding with MetalLB-owned IPs.

### Step 1c: Disable IPv6 (WAN side too)

UDM Settings → **Internet → Primary Connection → IPv6**: set to **Disabled**.

!!! warning "IPv6 has two settings — both matter"
    The per-VLAN IPv6 toggle (Step 1, item 7) only stops UDM from handing IPv6 to LAN clients. It does **not** stop UDM from accepting an IPv6 prefix from your ISP. Without disabling at WAN, clients could still be reachable over IPv6 via SLAAC even though you "disabled IPv6 on the LAN."

## Step 2: Connect devices to VLANs

VLAN networks are inert until devices are attached. Two paths: WiFi (SSID-to-VLAN binding) and wired (switch port profile).

### Step 2a: Create WiFi SSIDs

UniFi Network → **WiFi → Create New SSID** for each row in the SSID mapping table:

| SSID | Network | Notes |
|---|---|---|
| `home` | Trusted | personal devices |
| `home-iot` | IoT | Apple TV, smart home |

For each SSID: select the matching network from the **Network** dropdown, set a strong password (store in Vaultwarden), and enable **PMF (Protected Management Frames)** as Required or Optional — prevents deauth attacks on modern clients.

### Step 2b: Configure wired switch port profiles

Every wired device needs its switch port assigned to the right network. By default, switch ports are on the Default LAN — meaning a desktop you cable up will land on `10.0.0.0/24` regardless of which VLAN you intended.

UniFi → **UniFi Devices → [your switch] → Ports**. For each relevant port click into it and set **Native VLAN / Network** to the right VLAN:

| Port use | Native network |
|---|---|
| Home desktop | Trusted (VLAN 10) |
| ruby, emerald, topaz, amethyst (Turing Pi 2 ethernet) | Lab (VLAN 20) |
| UDM uplink, AP uplink, downstream switches | Default (or a trunk allowing all VLANs) |

!!! note "Access vs trunk ports"
    A port set to a single Native network is in **access mode** — it carries one untagged VLAN. Use this for endpoint devices (desktop, cube node, IoT device).

    A port that needs to carry multiple VLANs (uplink to another switch, port for a virtualization host with tagged VLAN interfaces) should be left on Default with **all VLANs allowed** — this is **trunk mode** and how the AP gets all SSIDs/VLANs to broadcast.

### Step 2c: Apple TV DHCP reservation

Connect the Apple TV to the `home-iot` SSID. After it gets an initial DHCP lease, find it in **UniFi → Clients → Apple TV → Settings → Fixed IP Address** and set to `10.0.30.10` (per the Static IP allocations table). Reboot the Apple TV to apply.

## Step 3: Firewall with Policy Engine (Zone-Based)

UniFi OS 4+ moved the firewall from the legacy `LAN IN / LAN OUT` rule list into the **Policy Engine**, which uses zones and a zone matrix. The model:

- **Zones** group networks. Built-in zones include Internal, External (WAN), Gateway, VPN, Hotspot/Guest, DMZ.
- **Zone Matrix** sets the default action (Allow / Block) between every pair of zones.
- **Policies** override the matrix for specific flows (source zone, destination zone, ports).

The big win: **default-deny between custom zones is built in.** You list only what's allowed; everything else is dropped.

### Step 3a: Create custom zones

UniFi → **Settings → Policy Engine → Zone Matrix → Add Zone** (UI labels may vary slightly between Network application versions — look for zone configuration under Policy Engine). Create three zones and assign each VLAN:

| Zone | Networks |
|---|---|
| Trusted | VLAN 10 |
| Lab | VLAN 20 |
| IoT | VLAN 30 |

Leave the Default LAN in the built-in **Internal** zone — it's the management plane and should reach the custom zones by default.

### Step 3b: Set the zone matrix defaults

In the Zone Matrix, set the default action for each zone pair as below. Intra-zone (a zone to itself) is implicitly Allow — devices on the same VLAN talk freely. Cells marked `Block*` are overridden by a specific allow policy in Step 3c.

| From ↓ / To → | Internal | Trusted | Lab | IoT | External |
|---|---|---|---|---|---|
| **Internal** | (intra) | Allow | Block | Block | Allow |
| **Trusted** | Allow | (intra) | Block* | Allow | Allow |
| **Lab** | Block | Block | (intra) | Block | Allow |
| **IoT** | Block | Block | Block* | (intra) | Allow |
| **External** | Block | Block | Block | Block | (n/a) |

The asymmetry around Internal is deliberate: Trusted reaches Internal (so you can admin the switch, AP, and Turing Pi BMC from your desktop), but Lab and IoT cannot (cluster workloads and smart bulbs have no business reaching the management plane). Internal → Trusted stays Allow so UDM/AP firmware update flows that initiate from the controller side still work.

### Step 3c: Add allow policies (the explicit exceptions)

UniFi → **Policy Engine → Policies → Create Policy**. Each policy overrides a zone-matrix Block for a specific flow:

| Name | From | To | Action | Protocol | Ports |
|---|---|---|---|---|---|
| trusted-to-lab-services | Trusted | Lab | Allow | TCP | `80, 443, 22, 6443` |
| iot-to-home-assistant | IoT | Lab | Allow | TCP | `8123` |

That's it. Two policies plus the zone matrix replace what used to be 6–8 rules in the legacy UI. Anything not in a policy falls through to the matrix default (Block for the inter-zone pairs above).

!!! note "Why the matrix-then-policies model is better"
    With legacy `LAN IN` rules, the implicit "allow everything else" meant you had to remember to add catch-all drops for every flow you wanted restricted. Forget one and you had a silent hole. The zone matrix makes the default explicit: Block is the baseline, Allow requires a policy. Audit becomes "list the policies" instead of "spot the missing drop."

!!! note "UDM is in the data path for inter-VLAN traffic"
    Because UDM is the L3 router between VLANs, every Trusted → Lab service request flows through UDM. Performance is fine for homelab scale (gigabit class), but it means firewall policies apply on every request and UDM CPU sees the load. The alternative (MetalLB in BGP mode with UDM peering) eliminates this but requires UDM Pro/SE with BGP enabled.

## Step 4: Reaching UDM

UDM accepts management on every VLAN gateway IP by default. Where you connect from determines the IP.

| Where you are | URL | Requires |
|---|---|---|
| Default LAN | `https://10.0.0.1` | nothing |
| Trusted VLAN 10 | `https://10.0.10.1` | nothing — UDM listens on the VLAN gateway by default |
| Lab VLAN 20 | `https://10.0.20.1` | UDM management enabled on Lab (default on) |
| IoT VLAN 30 | `https://10.0.30.1` | UDM management enabled on IoT (consider disabling, see warning) |
| Off-network (anywhere) | `https://10.0.0.1` over Tailscale | Tailscale connected; default LAN route accepted |

!!! warning "Consider disabling UDM management on IoT"
    UniFi → Networks → IoT → Advanced → "Allow this network to use the Local Management Service" — turn off. There is no reason a smart bulb should be able to reach the gateway UI.

## Step 5: Tailscale Subnet Router

Install Tailscale on two cluster nodes (ruby + emerald) so the subnet router survives reboots. Configure ACLs so only your devices can reach the advertised subnets.

!!! warning "Requires ruby and emerald"
    This step needs the first two cluster nodes booted and reachable. If you have not finished Runbook 3 yet, complete Steps 1–4 of this runbook now (UDM, devices, firewall, UDM access) and return for Step 5 after the cluster nodes are flashed.

### Step 5a: Generate a pre-authorized auth key

In the Tailscale admin console → **Settings → Keys → Generate auth key**:

- **Reusable:** on (you will use it on two nodes)
- **Ephemeral:** off (subnet routers must persist across reboots)
- **Pre-approved:** on (machine joins without manual approval)
- **Tags:** `tag:homelab-router` (referenced from ACLs in Step 5d)
- **Expiration:** 24h (long enough for the install session, short enough that a leaked key dies fast)

Copy the `tskey-auth-...` value. Don't commit it. Store in Vaultwarden if you need it past today.

### Step 5b: Install on ruby

```bash
curl -fsSL https://tailscale.com/install.sh | sh

sudo tailscale up \
  --authkey=tskey-auth-XXXXX \
  --advertise-routes=10.0.0.0/24,10.0.10.0/24,10.0.20.0/24 \
  --advertise-tags=tag:homelab-router \
  --ssh
```

`--ssh` enables Tailscale SSH on this node — SSH that authenticates via your Tailnet identity, no separate key management. Optional but very convenient when admin-ing from a phone.

### Step 5c: Install on emerald for failover

Same auth key (it's reusable). Same routes. Tailscale will pick one router as primary and switch to the other automatically if the primary goes offline. Brief (~30s) blip during failover.

```bash
curl -fsSL https://tailscale.com/install.sh | sh

sudo tailscale up \
  --authkey=tskey-auth-XXXXX \
  --advertise-routes=10.0.0.0/24,10.0.10.0/24,10.0.20.0/24 \
  --advertise-tags=tag:homelab-router \
  --ssh
```

!!! note "Why a second subnet router matters"
    Ruby will reboot regularly — k3s upgrades, kernel updates, Ansible runs. Without a failover router, every reboot kills your remote access to the entire homelab until ruby comes back up. The failover takes five minutes to set up now and removes a sharp foot-gun forever.

### Step 5d: Configure Tailscale ACLs

Out of the box every Tailnet member can reach every advertised route. With three subnets exposed (default, trusted, lab), an ACL is what scopes "who can reach what."

Tailscale admin console → **Access Controls → Edit file**. Replace the default Allow-All with:

```json
{
  "tagOwners": {
    "tag:homelab-router": ["autogroup:owner"]
  },
  "acls": [
    {
      "action": "accept",
      "src": ["autogroup:owner"],
      "dst": ["10.0.0.0/24:*", "10.0.10.0/24:*", "10.0.20.0/24:*"]
    }
  ]
}
```

`autogroup:owner` is "devices owned by the Tailnet owner" — i.e., yours. Any future device added for someone else (family, friend) gets zero access until you write an explicit ACL for them.

### Step 5e: Approve advertised routes

Pre-authorized auth keys join the Tailnet without manual approval, but **subnet routes still need per-machine approval**. In the admin console → Machines → ruby → **Edit route settings** → check all three subnets → Save. Repeat for emerald.

Install Tailscale on your phone, laptop, or any device you want to reach the homelab from.

!!! note "MagicDNS — leave off for now"
    MagicDNS makes Tailscale's `100.100.100.100` resolver handle DNS for Tailscale-connected clients. If you later run AdGuard in-cluster as your home DNS, the two resolvers can race for clients on the Tailnet while at home. Leave MagicDNS disabled until you've decided which DNS layer wins — or scope it later with Tailscale's "Restrict to domain" feature.

!!! note "Why IoT is intentionally excluded from advertised routes"
    Advertising the IoT subnet would let a compromised Tailscale client (browser zero-day, bad npm install) pivot into IoT devices. IoT firmware is rarely patched and a great place for an attacker to hide persistence. The marginal usability gain (direct IP access to smart devices from afar) is small — Home Assistant exposes IoT control via a web UI that you can reach over Tailscale through the Lab subnet.

!!! note "UDM firewall and Tailscale"
    Tailscale uses outbound UDP 41641 (or falls back to a relay over 443/TCP). UDM's default outbound is open, so this works without rules. If you locked outbound down on Lab VLAN, explicitly allow UDP 41641 outbound from ruby and emerald.

### Alternative: WireGuard on UDM

If you prefer fully self-hosted remote access without a third-party coordination server:

1. UDM → **Settings → VPN → VPN Server**.
2. Enable WireGuard and configure a listening port.
3. Create client profiles.

Tailscale is easier (NAT traversal handled, no port forward needed). WireGuard on UDM is fully self-hosted but harder to set up and requires inbound port-forwarding through your ISP. Both work.

## Verification

- [ ] Four networks visible in UniFi: Default, Trusted, Lab, IoT — each with the correct subnet and DHCP range from the Network Plan
- [ ] DHCP exclude range `10.0.20.200–.250` configured on Lab
- [ ] mDNS enabled on Trusted and IoT, disabled on Lab and Default
- [ ] IPv6 disabled on both WAN and every VLAN
- [ ] SSIDs `home` (Trusted) and `home-iot` (IoT) created and assigned
- [ ] Wired ports for home desktop and all four cluster nodes set to the right Native VLAN
- [ ] Policy Engine: three custom zones (Trusted, Lab, IoT), zone matrix Block-by-default between custom pairs, two allow policies (trusted→lab services, iot→HA)
- [ ] From a Trusted device: `https://10.0.10.1` loads UDM UI
- [ ] From a Trusted device: `curl https://10.0.20.10:6443` reaches ruby k3s API
- [ ] From a Trusted device: arbitrary high-port connection to ruby (e.g. `nc -zv 10.0.20.10 9999`) is **refused** — proves the policy restricts to listed ports, not any
- [ ] From an IoT device: `ping 10.0.20.10` fails (no policy allows it)
- [ ] From an IoT device: `curl http://<HA-IP>:8123` succeeds (iot-to-HA policy)
- [ ] From a phone on `home` SSID: AirPlay from Photos / Music finds Apple TV in the picker and casts successfully — proves mDNS reflector is working
- [ ] From a Trusted device with Tailscale connected via cellular: `ping 10.0.20.10` over Tailscale works
- [ ] `tailscale status` on ruby AND emerald shows advertised routes accepted
- [ ] Tailscale admin console: ACL file in place, only `autogroup:owner` has access to advertised subnets
- [ ] Failover test: `sudo tailscale down` on ruby — verify remote access still works via emerald within ~30s, then `tailscale up` ruby to restore
- [ ] `tailscale status` on your phone/laptop shows ruby reachable
