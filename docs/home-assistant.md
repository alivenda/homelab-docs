# Home Assistant

!!! success "Status — Live"
    Live as Home Assistant OS in a Proxmox VM on slate (the Mac mini).

Central hub for all smart home devices.

| | |
|---|---|
| **Difficulty** | Beginner |
| **Time Estimate** | 30–60 minutes |
| **Runs On** | NAS, dedicated device, or k3s (host networking) |

!!! tip "Three deployment targets, in increasing capability"
    1. **docker-compose on the NAS (this runbook)** — works for users who don't need Z-Wave/Zigbee/ESPHome add-ons.
    2. **Home Assistant OS on a dedicated mini-PC or VM** — the production answer; full add-on support.
    3. **Helm chart on k3s with hostNetwork** — only if you really want everything in the cluster and you've thought through how mDNS/SSDP discovery will work behind cluster networking.

    The compose path below is the simplest of the three.

!!! warning "Container install limits"
    The Container install shown here does NOT support add-ons like Z-Wave JS UI or ESPHome. If you rely on add-ons, use Home Assistant OS on a dedicated device or VM.

!!! note "What this homelab runs"
    This homelab uses **option 2**: Home Assistant OS as a Proxmox VM on **slate** (a repurposed Late-2014 Mac mini) at `10.0.20.21`, so Z-Wave/Zigbee/ESPHome add-ons and the backup pipeline in [Backups](backups.md) all work. The compose path below is kept as the simplest general option. HAOS-on-Proxmox gotcha: create the VM as `q35` with OVMF (UEFI) and **uncheck "Pre-Enroll keys" on the EFI disk**, or HAOS won't boot.

## Install

```yaml
# /volume1/docker/homeassistant/docker-compose.yml
services:
  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    container_name: homeassistant
    restart: always
    privileged: true
    network_mode: host
    environment:
      TZ: America/New_York
    volumes:
      - ha-config:/config

volumes:
  ha-config:
```

```bash
docker compose up -d
```

!!! tip "Network isolation"
    Put smart devices on the IoT VLAN and only allow Home Assistant to cross VLANs. This isolates insecure IoT devices.

## Traefik HTTPRoute (optional)

Same pattern as Immich — front the Home Assistant host (here, slate's HAOS VM at `10.0.20.21`) with a selector-less `Service` + a manual `EndpointSlice` pointing at its IP, then attach an `HTTPRoute` for `ha.yourdomain.com`. See [Immich's runbook](immich.md) for the manifest shape and [Deploying an App](apps-deploy-pattern.md) for the routing pattern.

!!! warning "Home Assistant rejects reverse-proxied requests by default"
    Set `http.use_x_forwarded_for: true` and `http.trusted_proxies:` in HA's `configuration.yaml`, or HA returns `400 Bad Request` behind Traefik. For an **off-cluster** backend like slate's VM, Traefik's egress to the LAN is SNAT'd to the k3s node, so HA sees the **node IP** — trust the node IPs (`10.0.20.10`–`10.0.20.13`), not the pod CIDR. (Only an *in-cluster* HA would trust the pod CIDR `10.42.0.0/16`.)

## Verification

- [ ] On the NAS:

    ```bash
    docker compose ps    # homeassistant container Running
    ```

- [ ] `http://<nas-ip>:8123` (or `https://ha.yourdomain.com` via Traefik) loads the HA onboarding flow.
- [ ] At least one integration discovered automatically (router, smart bulbs on the IoT VLAN).
- [ ] Restart cleanly: `docker compose restart homeassistant` — returns to UI in under 30 seconds.

!!! note "HAOS-on-slate checks"
    For the deployment this homelab runs: the VM boots to onboarding at `http://10.0.20.21:8123`, `https://ha.yourdomain.com` loads via Traefik, and a manual backup reaches Garage after the NAS sync ([Backups](backups.md)).
