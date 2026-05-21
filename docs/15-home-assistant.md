# Runbook 15: Home Assistant

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

## Traefik IngressRoute (optional)

Same pattern as Immich — use an ExternalName Service pointing at the NAS, then add an IngressRoute for `ha.yourdomain.com`.

## Verification

- [ ] On the NAS:

    ```bash
    docker compose ps    # homeassistant container Running
    ```

- [ ] `http://<nas-ip>:8123` (or `https://ha.yourdomain.com` via Traefik) loads the HA onboarding flow.
- [ ] At least one integration discovered automatically (router, smart bulbs on the IoT VLAN).
- [ ] Restart cleanly: `docker compose restart homeassistant` — returns to UI in under 30 seconds.
