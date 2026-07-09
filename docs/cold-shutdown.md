# Cold Shutdown & Storage

Powering down the **entire stack** — cluster, NAS, Home Assistant host, network gear — for **more than a day**: a house move, extended travel, electrical work, long-term storage.

| | |
|---|---|
| **Difficulty** | Beginner–Intermediate |
| **Time Estimate** | 1–2 hours down, 1–2 hours up (plus however long it stays dark) |
| **Runs On** | Everything |

An overnight power-down is just [Turing Pi's Planned Shutdown & Startup](turing-pi.md#shutdown); longer than that and the secondary effects start to bite — backup jobs silently miss their windows, RTC-less boards drift, and the Sealed Secrets key can cross its rotation boundary while the controller is off. Turing Pi owns the cluster-internal ordering (etcd snapshot, NFS clients before the NFS server); this page is the layer above it: final data captures while everything is still running, the cross-device ordering, transporting the hardware if it's moving, and a cold-start sequence plus verification checklist for the day it all comes back.

## Before you power anything down

### Make sure you can log in without the homelab

Vaultwarden runs *inside* the cluster, so it is offline for the entire outage. Anything you might need between power-off and full recovery must live in the externally-hosted password manager (see [Backups's keystone warning](backups.md#secrets-and-key-material-recovery)) — not only in Vaultwarden:

- ISP account credentials — especially if you're moving, since you'll be setting up the new WAN before anything self-hosted exists
- Router/controller admin login, BMC root, NAS admin, Proxmox root
- Tailscale account (handy for verifying remote access once back up)

Forgejo is also in the cluster — while it's down, your repos are reachable read-only via their GitHub mirrors.

### Final captures

Take these while everything is still running, in this order (cluster captures first, NAS syncs after, since the NAS jobs *pull from* the other devices):

1. **On-demand Velero backup** of the whole cluster:

    ```bash
    velero backup create pre-shutdown-$(date +%Y%m%d) --wait
    velero backup describe pre-shutdown-$(date +%Y%m%d) --details
    ```

    Gate on content, not the green `Completed`: the per-volume `PodVolumeBackups` in the describe output must show **non-zero bytes** for every app volume you care about.

2. **etcd snapshot** (also uploads to Garage):

    ```bash
    ssh dietpi@10.0.20.10 'sudo k3s etcd-snapshot save'
    ```

3. **Fresh Home Assistant backup**, then a final sync to Garage. Trigger the backup in HA (Settings → System → Backups → *Create backup*), wait for it to finish, then on the NAS:

    ```bash
    sudo systemctl start ha-backup-sync.service
    systemctl status ha-backup-sync.service   # both ExecStarts exited 0/SUCCESS
    ```

4. **Final database dump** on the NAS:

    ```bash
    sudo systemctl start postgres-backup.service
    ```

5. **Final NAS app-database syncs to Garage** — Immich and Audiobookshelf each write their own
   backups on a schedule; these units ship the latest to Garage so the cold-export below picks
   them up:

    ```bash
    sudo systemctl start immich-backup-sync.service
    sudo systemctl start audiobookshelf-backup-sync.service
    ```

    Immich's own database dump runs nightly at 02:00 — if you're powering down before then,
    take a fresh one first so the sync has something current to ship:

    ```bash
    docker exec -t immich_postgres pg_dumpall -c -U postgres \
      > /volume1/photos/backups/immich_pre-shutdown_$(date +%Y%m%d).sql
    ```

!!! warning "Powered down, your data and its backups share one box"
    Garage — the backup target for Velero, etcd, the databases, and HA — lives **on the NAS**. For the whole outage the originals and every backup sit in the same chassis: one truck if you're moving, one storage unit if it's going dark for a season. This is exactly the scenario [Backups's off-site TODO](backups.md) warns about. Cheap mitigation: sync the critical buckets to an external drive and keep it somewhere else (different bag, different car, different building).

    Every Garage key is scoped to its consumer's bucket, so no existing key can read them all — mint a temporary read-only export key, use it, delete it:

    ```bash
    # on the NAS
    docker exec garage /garage key create cold-export   # note the GK… ID + secret
    for b in velero etcd-snapshots postgres-backups ha-backups sealed-secrets-keys immich-backups audiobookshelf-backups; do
      docker exec garage /garage bucket allow --read $b --key cold-export
    done

    # point an rclone remote at it (type=s3, provider=Other,
    # endpoint=http://10.0.20.50:9000, region=us-east-1), then per bucket:
    for b in velero etcd-snapshots postgres-backups ha-backups sealed-secrets-keys immich-backups audiobookshelf-backups; do
      rclone sync coldexport:$b /mnt/usb/cold-shutdown/$b
    done

    docker exec garage /garage key delete --yes cold-export
    ```

    The sealing-key bucket and the password manager together can decrypt everything; the drive itself holds the keys only in their client-side-encrypted form.

## Shutdown order

Writers stop before the things they write to. Network gear goes last because everything else is managed *over* it.

1. **Home Assistant, then its host** — shut the HAOS VM down cleanly (HA: Settings → System → Hardware → Shutdown, or from the Proxmox UI), then shut down the Proxmox host itself.
2. **The cluster** — follow [Turing Pi's shutdown](turing-pi.md#shutdown) exactly: amethyst, emerald, ruby; *confirm all three have halted*; then topaz (the NFS server) last.
3. **The NAS** — only after the cluster is fully down (the etcd S3 upload and the sync jobs above are its last writers). Use the NAS UI's shutdown, or `sudo poweroff` over SSH.
4. **Network gear** — gateway, switch, APs. Their configuration persists on-device.
5. **UPS** — power it off; if it's being transported or stored long-term, disconnect the battery (tape exposed terminals). A jostled lead-acid battery shorting against a chassis is the one genuinely dangerous item in the load.

## Transport & storage notes

Skip this section if the hardware stays racked where it is.

- **Photograph the rack and label both ends of every cable** before unplugging anything. Future-you at the other end is a different, more tired person.
- The CM4s (eMMC), BMC microSD, and SATA SSD are solid-state — no special handling beyond normal padding. Check the BMC's microSD is seated before packing.
- **The NAS's spinning drives are the most shock-sensitive cargo.** Keep the unit upright and well padded. If the drive bays don't lock firmly, pull the drives, pack each padded and **labeled with its bay number**, and reinsert in the same order at the destination.
- For longer-term storage, keep the gear somewhere dry and temperature-stable — drives tolerate cold storage far better than humidity swings.
- The external drive from the bucket export above lives separately from the NAS, wherever the NAS ends up.

## Cold start

Reverse dependency order: network → NAS → Home Assistant host → cluster.

1. **Modem + gateway.** If you moved, the new ISP's WAN settings are the only thing that actually changed — LAN, VLANs, firewall rules, and DHCP reservations all persist on the gateway. Confirm a machine on the trusted VLAN gets an address and can reach the internet.

    !!! warning "Internet before cluster — the CM4s have no RTC"
        The compute modules boot with their clocks set to whenever they last shut down (or worse). Until chrony reaches an NTP server and steps the clock, expect x509 *"certificate is not yet valid"* noise and unhappy etcd. Bring the WAN up **before** powering the cluster, and verify time on each node once booted: `chronyc tracking` should show an offset in the millisecond range.

2. **NAS.** Power on, then confirm its services came back:

    ```bash
    sudo docker ps                       # garage + postgres containers running
    sudo docker exec garage /garage status
    systemctl list-timers                # postgres-backup + ha-backup-sync scheduled
    ```

3. **Home Assistant host.** Power on the Proxmox machine; the HAOS VM should auto-start (verify the VM's *Start at boot* option is set **before** the shutdown, not after).

4. **The cluster** — follow [Turing Pi's startup](turing-pi.md#startup): topaz first (NFS), then ruby (wait for `Ready`), then emerald and amethyst. ArgoCD reconciles the workloads on its own; give it a few minutes before touching anything.

    !!! tip "UFW may come back half-loaded"
        If `sudo ufw status` on a node fails with *"problem running ip6tables"*, the firewall is in a half-loaded state (empty IPv6 chains). Don't reboot or retry — reset it:

        ```bash
        sudo ufw --force disable && sudo ufw enable
        ```

    !!! note "Sealed Secrets may rotate its key at startup"
        The controller mints a new sealing key when the active one is **older than 30 days**, evaluated while running — so after a long downtime, expect a new key `Secret` moments after the controller starts. This is normal. The daily key-backup CronJob ships it to Garage within a day; verify it below.

## Post-restart verification

- [ ] All four nodes `Ready`, pods settle without crash-looping:

    ```bash
    kubectl get nodes && kubectl get pods -A | grep -v Running
    ```

- [ ] Clocks synced on every node: `chronyc tracking` offset in the millisecond range.
- [ ] `sudo ufw status` → `active` on all four nodes (see the half-load tip above).
- [ ] ArgoCD: every Application `Synced`/`Healthy`.
- [ ] Public endpoints reachable end-to-end: the blackbox `public-endpoints` Probe drives the `PublicEndpointDown` alert (fires to ntfy if any of the six public HTTPS hosts stay down for 5m), so a quiet inbox already means they're up. To spot-check directly, curl each — every one should return `200`:

    ```bash
    for h in auth argocd immich vault ha lldap; do
      printf '%s: ' "$h"; curl -so /dev/null -w '%{http_code}\n' "https://$h.yourdomain.com"
    done
    ```
- [ ] Velero: `velero backup-location get` → `Available`, and the next scheduled backup completes **with non-zero per-volume bytes**.
- [ ] etcd: the next scheduled snapshot lands in Garage.
- [ ] Tailscale: subnet routers advertise and connect (a new public IP is transparent to the tailnet).
- [ ] Certificates: `kubectl get certificate -A` all `Ready` (DNS-01 renewal needs the WAN, which is up by now).
- [ ] Home Assistant reachable at `ha.yourdomain.com`; automations resumed.
- [ ] If the downtime crossed a sealing-key rotation: **two** key Secrets exist —

    ```bash
    kubectl get secrets -n sealed-secrets -l sealedsecrets.bitnami.com/sealed-secrets-key
    ```

    — and the next day's key-backup dump in Garage contains both (decrypt it and check `jq '.items | length'`).
- [ ] End-to-end alerting still works: publish a test message through ntfy (see [ntfy](ntfy.md)) and confirm it reaches your phone.
