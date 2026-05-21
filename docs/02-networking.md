# Runbook 2: Network Segmentation and VPN

VLANs on UDM and remote access via Tailscale.

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Time Estimate** | 1–2 hours |
| **Runs On** | UDM + a Turing Pi node for Tailscale subnet routing |

## VLAN Architecture

- **VLAN 10 — Trusted:** Your PCs, phones, tablets. Full access.
- **VLAN 20 — Lab/Servers:** Turing Pi 2, NAS, Docker hosts.
- **VLAN 30 — IoT:** Smart home devices. Restricted to Home Assistant.

## Step 1: Create VLANs on UDM

1. UniFi Network app → Settings → Networks.
2. Create networks for each VLAN (`10.0.10.0/24`, `10.0.20.0/24`, `10.0.30.0/24`).
3. Enable DHCP for each VLAN.
4. Assign WiFi SSIDs to VLANs.

## Step 2: Firewall Rules

- Allow: VLAN 10 → VLAN 20 (all)
- Allow: VLAN 10 → VLAN 30 (all)
- Allow: VLAN 30 → VLAN 20 (only Home Assistant IP:8123)
- Block: VLAN 30 → VLAN 10 (all)
- Block: VLAN 20 → VLAN 10 (except established/related)

## Step 3: Tailscale (highest QoL win)

!!! warning "Requires cube01"
    Step 3 requires cube01 to exist. If you haven't finished Runbook 3 yet, do Steps 1–2 of this runbook now (UDM VLANs and firewall rules) and return for Step 3 after the cluster nodes are flashed and booted.

Install on cube01 and advertise your lab subnets:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --advertise-routes=10.0.0.0/24,10.0.10.0/24,10.0.20.0/24,10.0.30.0/24
```

Then:

- In the Tailscale admin console, approve the subnet routes.
- Install Tailscale on your phone/laptop.
- Optional: enable MagicDNS so node hostnames resolve over Tailscale.

!!! note "UDM firewall and Tailscale"
    Tailscale uses outbound UDP 41641 (or falls back to a relay over 443/TCP). UDM's default outbound is open, so this works without rules. If you locked outbound down on VLAN 20, explicitly allow UDP 41641 outbound from cube01.

### Alternative: WireGuard on UDM

1. Settings → VPN → VPN Server.
2. Enable WireGuard and configure listening port.
3. Create client profiles.

!!! tip
    Tailscale is easier (NAT traversal handled). WireGuard on UDM is fully self-hosted but harder to set up. Both work well.

## Verification

- [ ] In UniFi Network, three VLANs visible: 10/20/30, each with a DHCP scope, each on a separate WiFi SSID if applicable
- [ ] From a VLAN 10 device: `ping 10.0.0.60` (cube01 on VLAN 20) succeeds
- [ ] From a VLAN 30 device: `ping` to anything in VLAN 20 fails EXCEPT Home Assistant on port 8123
- [ ] From your laptop with Tailscale up: `ping 10.0.0.60` over Tailscale works

```bash
tailscale status   # on cube01 - should show advertised routes accepted
tailscale status   # on your laptop - should show cube01 reachable
```
