# Disaster Recovery

Something is dead and you're deciding what to do first. This page is **triage and sequencing only** — find your row in the table, follow the ordered checklist, and do the actual work in the linked runbooks. Nothing here duplicates a procedure that lives elsewhere.

| | |
|---|---|
| **Difficulty** | Varies by scenario — a dead workstation is ~20 minutes; total loss is days |
| **Step zero, always** | [Secrets and key-material recovery](backups.md#secrets-and-key-material-recovery) — nothing restores until you can decrypt |
| **Planned downtime instead?** | A move or long storage is not a disaster — use [Cold Shutdown](cold-shutdown.md) |
| **Last drilled** | Secrets chain (scenario 1 + the Garage path): 2026-07-17, [full pass](backups.md#drill-the-whole-chain-no-dead-machine-required) |

## What died?

| What's dead | What you see | Go to |
|---|---|---|
| Your workstation | The homelab runs fine; you just can't operate it | [Scenario 1](#scenario-1-your-machine) |
| emerald or amethyst | Node `NotReady`; its pods reschedule or go `Pending` | [Scenario 2](#scenario-2-an-agent-node-emerald-or-amethyst) |
| topaz | Node `NotReady` **and** every `nfs-storage` app hangs | [Scenario 3](#scenario-3-topaz-the-storage-node) |
| ruby | `kubectl` times out; the apps themselves still serve | [Scenario 4](#scenario-4-ruby-the-control-plane) |
| The NAS | Postgres-backed apps, Immich, Audiobookshelf down; backups have nowhere to land | [Scenario 5](#scenario-5-the-nas) |
| Everything on-site | Fire, theft, flood | [Scenario 6](#scenario-6-total-loss) |

!!! warning "Confirm the diagnosis before rebuilding anything"
    The symptoms overlap with plain network trouble. `kubectl` timing out means ruby **or** the path to it — `ping 10.0.20.10` and check an app URL before concluding the control plane is dead. Every app hanging on storage means topaz **or** the switch/VLAN between the nodes. Reflash only after the [static-IP table](networking.md#static-ip-allocations) says the box itself, not the network, is the problem.

## Scenario 1 — your machine

Nothing in the homelab is affected — this is about restoring your ability to operate it. This exact chain is the one drilled on 2026-07-17.

1. On the new or reinstalled machine, install the base tooling as each runbook calls for it — `git` and `sops`/`age` are the only two you need before anything else works.
2. Restore the age key from your (externally-hosted) password manager — [Backups, Step 1](backups.md#step-1-restore-the-age-private-key).
3. Clone the five repos from Forgejo (`git.yourdomain.com` — it's unaffected); layout and remotes are in [Git Foundation](git.md).
4. Confirm you can decrypt one file from each repo class — [Backups, Step 2](backups.md#step-2-confirm-you-can-decrypt-the-repos).
5. Pull the kubeconfig from ruby — [Kubernetes, Step 0](kubernetes.md#step-0-get-kubectl-working-on-your-machine).
6. Verify: `kubectl get nodes` shows four `Ready` nodes, and `tofu plan` runs clean against the restored state backend.

## Scenario 2 — an agent node (emerald or amethyst)

Stateless pods reschedule on their own; what makes an agent-node death more than a long reboot is **`local-path` data**, which lives on the dead node's eMMC. emerald is the designated app-state node (see [the local-path tier](storage-architecture.md#the-local-path-tier)), so losing emerald means restoring SQLite app data from Velero; losing a node with no `local-path` PVCs is just reflash-and-rejoin.

1. Check what the node was holding before you rebuild: `kubectl get pv -o wide` — any `local-path` PVs on the dead node mark the apps you'll restore in step 4.
2. Reflash DietPi on the module — [Turing Pi, Step 3](turing-pi.md#step-3-flash-dietpi-os-to-all-4-modules).
3. Re-run `site.yml` from [Ansible](ansible.md) — it reinstalls the **pinned** k3s agent, rejoins with the same token, and re-applies the node's labels from `inventory.yml` (including `app-state=true`). Confirm: node `Ready`, labels present in `kubectl get nodes --show-labels`.
4. If the node held `local-path` PVCs, their data died with the eMMC and the PVC objects that survived in etcd are empty shells. Velero skips resources that already exist, so remove the shells first, then restore per app:

    ```bash
    velero backup get                                   # newest daily-cluster-…
    kubectl delete pvc -n <app> <pvc-name>              # the PV goes with it
    velero restore create --from-backup daily-cluster-<date> --include-namespaces <app>
    velero restore describe <restore-name> --details    # every volume Completed, non-zero bytes
    ```

5. Verify content, not exit codes: open each restored app and check real data, per [Test your restores](backups.md#test-your-restores). Finish with ArgoCD all `Synced`/`Healthy`.

## Scenario 3 — topaz, the storage node

Every `nfs-storage` PVC is served from the single SSD in topaz — the storage SPOF named in the [durability model](storage-architecture.md#durability-model). Apps hang until it's back; data in the Postgres tier is safe on the NAS. First establish which of the two failures you have: a dead **module** (SSD almost certainly intact) or a dead **SSD**.

1. Reflash the module — [Turing Pi, Step 3](turing-pi.md#step-3-flash-dietpi-os-to-all-4-modules) — and reconnect the SSD ([Step 5](turing-pi.md#step-5-prepare-sata-ssd-on-node-3-topaz-node-3) covers the physical side; don't run its format commands on a disk with data).
2. Re-run `site.yml` from [Ansible](ansible.md). The NFS play is guarded — it only formats a **blank** disk — so on a surviving SSD it just remounts `/data`, re-exports it, and rejoins the agent.
3. If the SSD survived: the export is back and the data with it. Pods that were hanging on stale NFS mounts may need a kick — `kubectl rollout restart` anything still stuck, then confirm apps serve real data.
4. If the SSD is dead: replace it, let `site.yml` format and export the empty disk, then restore every `nfs-storage` app from Velero using the delete-the-shells-first flow from [scenario 2](#scenario-2-an-agent-node-emerald-or-amethyst), step 4 — Tier 1 apps from the [strategy table](backups.md#strategy-the-tiers) first.
5. Verify per [Test your restores](backups.md#test-your-restores): app content present, monitoring (which also lives on topaz) back, ArgoCD green.

## Scenario 4 — ruby, the control plane

The data plane keeps serving — apps stay up, you just can't deploy, scale, or restart anything. Don't power-cycle the agents while ruby is down; their workloads are fine. The full procedure with commands is [Kubernetes, Step 11](kubernetes.md#step-11-restore-procedure-when-ruby-dies); the order:

1. Reflash the module — [Turing Pi, Step 3](turing-pi.md#step-3-flash-dietpi-os-to-all-4-modules).
2. Re-run `site.yml` from [Ansible](ansible.md) — pinned k3s server install plus the `etcd-s3` config the restore reads.
3. Restore etcd from the newest Garage snapshot with `--cluster-reset` — [Step 11](kubernetes.md#step-11-restore-procedure-when-ruby-dies) has the exact commands.
4. Agents rejoin on their own (same token, same server IP). Run Step 11's post-restore checks: nodes `Ready`, sealed-secrets signing keys present, ArgoCD green.

!!! note "The restored state is up to 12 hours old"
    Snapshots run every 12 h. ArgoCD's self-heal replays anything that changed in Git since the snapshot; anything applied imperatively with `kubectl` after it is gone.

## Scenario 5 — the NAS

The NAS holds the entire Garage store (every backup bucket **and** the Terraform state), the shared Postgres tier (live databases), Immich (app, its own database, and the photo originals), Audiobookshelf, and the sync timers. Which recovery you're in depends entirely on the drives.

### Drives survived (chassis, PSU, or board failure)

1. Repair or replace the unit; reinsert the drives in their original bay order.
2. Give it back its static IP (`10.0.20.50` — [static-IP table](networking.md#static-ip-allocations)).
3. Start the Docker stacks: Garage ([stand up Garage](backups.md#stand-up-garage) is the reference), Postgres ([NAS PostgreSQL](nas-postgres.md#stand-up-the-server)), [Immich](immich.md), [Audiobookshelf](apps-catalog.md#audiobookshelf).
4. A UGOS reinstall resets host-level systemd units — recreate the sync timers from [the rclone → Garage pattern](backups.md#nas-side-sync-jobs-the-rclone-garage-pattern) and confirm every timer in [What runs when](backups.md#what-runs-when) shows a next run.
5. Verify: cluster apps that use the Postgres tier recover on their own once the server answers; `docker exec -ti garage /garage status` healthy; the next night's backups all land.

### Drives lost

!!! danger "Accepted data loss until off-site backup exists"
    Everything whose only copies lived on the NAS is gone: the entire Garage backup history, the shared Postgres databases **and** their dumps, and every Immich original not still on a device. This is exactly the gap the [off-site TODO](backups.md#what-runs-when) exists to close.

1. Rebuild the NAS: fresh UGOS, static IP, then [stand up Garage](backups.md#stand-up-garage) and re-run the bucket/key/allow trio for every consumer.
2. The re-minted keys are new credentials — update every consumer that stored the old ones: `etcd_s3_*` in the `homelab-ansible` SOPS file, Velero's SealedSecret, the NAS `rclone.conf` remotes, and the Terraform state backend (state is gone — re-import the live Cloudflare records before the next `tofu apply`).
3. Stand the Postgres tier back up and re-provision each app's database and role ([NAS PostgreSQL](nas-postgres.md#provisioning-a-database-for-an-app)); apps recreate their schemas on first start, with empty tables.
4. Redeploy [Immich](immich.md) and [Audiobookshelf](apps-catalog.md#audiobookshelf); re-upload photos from devices; let Audiobookshelf re-scan the (also lost) media you restore or re-rip.
5. Recreate the sync timers as in the drives-survived path, then force a full backup cycle immediately — don't wait for the schedules.

## Scenario 6 — total loss

Fire, theft, flood — everything on-site is gone. What survives is exactly what lives off-site today: the IaC (GitHub mirrors), the secrets chain (your externally-hosted password manager plus the encrypted files in the repos), and these docs. All *data* is gone per the [scenario 5 danger box](#drives-lost) — this rebuild restores the services, and devices restock what they still hold.

1. Confirm the keystone: you can get into your password manager — [the single root of trust](backups.md#the-single-root-of-trust).
2. On a new machine, clone the five repos from the **GitHub mirrors** — Forgejo lived in the cluster; the mirror wiring is in [Git Foundation](git.md#as-built-forgejo-primary-agit-prs-gitops-over-ssh).
3. Restore the age key and confirm decrypts — [Backups, Steps 1–2](backups.md#step-1-restore-the-age-private-key).
4. Network first: if the gateway survived, its config did too; otherwise rebuild it from [Networking](networking.md).
5. NAS, Garage, and the Postgres tier — the [scenario 5 drives-lost list](#drives-lost).
6. Flash the nodes ([Turing Pi](turing-pi.md)), bootstrap with [Ansible](ansible.md), bring up k3s and its core per [Kubernetes](kubernetes.md).
7. Wire ArgoCD to the **GitHub mirror** of `homelab-manifests` using the [day-zero pattern](git.md#step-7-wire-argocd-to-homelab-manifests) — the as-built wiring points at Forgejo, which doesn't exist again yet.
8. Restore the Sealed Secrets signing keys before the apps sync — [Backups, Step 3](backups.md#step-3-restore-the-sealed-secrets-signing-keys-cluster-rebuild-only). The Garage dump died with the NAS, so the day-zero fallback is all you have: it unlocks only secrets sealed before the first rotation. Everything sealed after it must be **re-created from its source** (password manager, provider dashboards) and re-sealed.
9. Rebuild DNS: `tofu apply` with the restored Cloudflare credentials — the state backend was on the NAS, so re-import the records the apply would otherwise duplicate.
10. Once Forgejo is redeployed, push the five repos into it and flip ArgoCD back to the [as-built SSH wiring](git.md#gitops-argocd-pulls-from-forgejo-over-ssh).
11. Run the full [post-restart verification](cold-shutdown.md#post-restart-verification), then force a complete backup cycle so the new build is protected from day one.

## After any recovery

- Run the [post-restart verification](cold-shutdown.md#post-restart-verification) — it's scenario-agnostic and catches the quiet failures (clock skew, half-loaded UFW, stale certs).
- Force fresh backups immediately instead of waiting for the schedules, and confirm them with the [Backups verification list](backups.md#verification).
- While the incident is fresh: if any step here didn't match reality, fix this page in the same PR as the postmortem note.
