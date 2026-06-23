# Ansible

Replace the manual per-node steps in Turing Pi with a single Ansible playbook. Provision all 4 CM4 nodes (or rebuild any one) in minutes.

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Time Estimate** | 2–3 hours upfront, saves hours forever |
| **Runs On** | Your machine (the Ansible control host) |
| **Replaces** | Manual setup steps in Turing Pi + k3s install |
| **DevOps Skills** | Configuration management, idempotency, infrastructure as code |

## Why Ansible (and Not Terraform)

Terraform shines for cloud APIs (creating EC2 instances, RDS databases, S3 buckets). For bare-metal homelabs, the actual creation step is physical (you put the CM4 in the slot). Ansible is the right tool for what comes after: making the OS look identical across all 4 nodes.

## Module Preferences: Native Modules over `shell:`

Ansible has native modules for most common ops. Reach for `shell:` only when no module covers the operation (e.g., piped install scripts like k3s). Native modules give you idempotency for free, structured error reporting, and check-mode (`--check`) support — three things `shell:` cannot.

Quick reference for tasks in this guide:

| Operation | Prefer this module | Notes |
|---|---|---|
| Install apt packages | `ansible.builtin.apt` | Use `update_cache: true` once per play |
| Add apt repo / key | `ansible.builtin.apt_repository` + `apt_key` | `apt_key` deprecated on 22.04+ — use `deb822_repository` |
| Load kernel module | `community.general.modprobe` | Idempotent; also handles persistence |
| Manage systemd unit | `ansible.builtin.systemd_service` | `state` + `enabled` in one task |
| Apply kubectl manifest | `kubernetes.core.k8s` | `state: present/absent`, `definition:` inline or `src:` |
| Helm release | `kubernetes.core.helm` | `release_name`, `chart_ref`, `values` + `release_state` |
| Wait for k8s resource | `kubernetes.core.k8s_info` | Loop with `until:` until ready |
| Template a config file | `ansible.builtin.template` | Jinja2 with handlers for reload |
| Reboot the host | `ansible.builtin.reboot` | Built-in wait-for-reconnect handling |

The two k3s install tasks in Step 8 of this runbook intentionally use `ansible.builtin.shell`. The k3s upstream install is a curl-piped script — wrapping it in a more idiomatic module would not improve correctness and would obscure the canonical install path. Use `creates: /usr/local/bin/k3s` on the task to keep it idempotent.

!!! tip "If you run ansible-lint"
    The k3s install tasks above will trip `command-instead-of-module` on ansible-lint's `basic` profile or higher. The curl-pipe is the canonical upstream install path, not an oversight — scope the exception narrowly with `# noqa: command-instead-of-module` on each install task rather than disabling the rule globally.

!!! tip
    Add the `kubernetes.core` collection to your `requirements.yml` before Kubernetes: `ansible-galaxy collection install kubernetes.core`. The `kubeconfig:` parameter on `kubernetes.core.k8s` lets you run cluster ops from your machine against the remote API.

## Step 1: Install Ansible on Your Machine

Ansible is agentless — it runs locally and talks to nodes over SSH. No agent install on the CM4s.

```bash
# macOS
brew install ansible
# Ubuntu / Debian
sudo apt install -y ansible
# Arch / CachyOS
sudo pacman -S ansible

ansible --version
```

## Step 2: SSH Key Distribution

Turing Pi's `dietpi.txt` sets each node's static IP (`AUTO_SETUP_NET_USESTATIC=1`) and hostname at first boot, so the nodes come up directly on their planned addresses (`10.0.20.10–.13`) — there's no DHCP-to-static transition to manage. A matching DHCP reservation in UDM → Clients is optional belt-and-suspenders.

Copy your key to the **`dietpi`** user — DietPi's default admin account. Ansible logs in as `dietpi` and escalates with `sudo` rather than logging in as root:

```bash
ssh-keygen -t ed25519 -C "ansible@homelab"   # If you don't have one
for ip in 10 11 12 13; do
  ssh-copy-id dietpi@10.0.20.$ip
done
```

!!! tip "Harden SSH once the key works"
    After key auth works for `dietpi`, disable root login and password auth on each node — set `PermitRootLogin no` and `PasswordAuthentication no` in `/etc/ssh/sshd_config`, then `sudo systemctl restart ssh`. Key-only, non-root is the baseline before exposing anything.

## Step 3: Project Structure

```
homelab-ansible/
├── inventory.yml
├── ansible.cfg
├── requirements.yml
├── site.yml          # all plays in one file
└── secrets/
    └── secrets.sops.yaml   # sops-encrypted; safe to commit
```

For a 4-node cluster, a single `site.yml` containing all plays keeps things readable in one screen. If the file grows past ~200 lines or you frequently need to re-run just one phase (and `--tags` / `--start-at-task` aren't enough), split into `playbooks/bootstrap.yml`, `playbooks/k3s-server.yml`, etc., and add a top-level `site.yml` of `import_playbook:` lines.

### Collections (`requirements.yml`)

The bootstrap and NFS playbooks pull from outside `ansible.builtin`, and `community.sops` provides the `k3s_token` lookup (Step 10). Pin them in `requirements.yml`:

```yaml
---
collections:
  - name: ansible.posix
    version: ">=2.0.0,<3.0.0"
  - name: community.general
    version: ">=13.0.0,<14.0.0"
  - name: kubernetes.core
    version: ">=6.0.0,<7.0.0"
  - name: community.sops
    version: ">=2.3.0,<3.0.0"
```

Install:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Step 4: Inventory (`inventory.yml`)

```yaml
all:
  hosts:
    ruby:
      ansible_host: 10.0.20.10
      node_role: server
      emmc_size: 32
    emerald:
      ansible_host: 10.0.20.11
      node_role: agent
      emmc_size: 32
    topaz:
      ansible_host: 10.0.20.12
      node_role: agent
      emmc_size: 16
      nfs_server: true
    amethyst:
      ansible_host: 10.0.20.13
      node_role: agent
      emmc_size: 16
  vars:
    ansible_user: dietpi
    ansible_python_interpreter: /usr/bin/python3
    k3s_token: "{{ (lookup('community.sops.sops', inventory_dir + '/secrets/secrets.sops.yaml') | from_yaml).k3s_token }}"
    k3s_server_ip: 10.0.20.10

k3s_server:
  hosts:
    ruby:

k3s_agents:
  hosts:
    emerald:
    topaz:
    amethyst:
```

!!! tip "The secret resolves at runtime"
    `k3s_token` is pulled from the sops-encrypted `secrets/secrets.sops.yaml` by the `community.sops` lookup and decrypted with your age key. Ansible only resolves Jinja at task-execution time, so create the secret (Step 10) before running `ansible-playbook` in Step 11 — otherwise the lookup fails with a missing-file or decrypt error.

## Step 5: `ansible.cfg`

```ini
[defaults]
inventory = inventory.yml
host_key_checking = True
retry_files_enabled = False
stdout_callback = default
callback_result_format = yaml
```

!!! note "Why not `stdout_callback = yaml`"
    community.general 12.0 removed the standalone `yaml` callback. The built-in `default` callback with `callback_result_format: yaml` gives the same readable multi-line output. There's no `vault_password_file` — sops + age handles secrets (Step 10).

## Step 6: Bootstrap Playbook

Steps 6–8 each show one play. Append them in order into `site.yml` at the repo root — that one file is the whole playbook for the cluster.

```yaml
- hosts: all
  become: true
  tasks:
    # Hostname + static IP come from dietpi.txt at first boot; Ansible owns config only.
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Install base packages
      ansible.builtin.apt:
        name: [iptables, nfs-common, curl, jq]
        state: present

    - name: Ensure cgroups enabled in cmdline.txt
      ansible.builtin.lineinfile:
        path: /boot/firmware/cmdline.txt
        backrefs: yes
        regexp: '^(.*?)(\s*cgroup_enable=cpuset.*)?$'
        line: '\1 cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1'
      register: cmdline

    - name: Reboot if cmdline changed
      ansible.builtin.reboot:
      when: cmdline.changed

    - name: Configure /etc/hosts
      ansible.builtin.blockinfile:
        path: /etc/hosts
        block: |
          10.0.20.10 ruby
          10.0.20.11 emerald
          10.0.20.12 topaz
          10.0.20.13 amethyst
```

!!! note "DietPi owns node identity — Ansible sets neither hostname nor IP"
    On DietPi the hostname (`AUTO_SETUP_NET_HOSTNAME`) and static IP (`AUTO_SETUP_NET_USESTATIC`) are set from `dietpi.txt` at first boot (Turing Pi Step 3), and the network is managed by `ifupdown` (`/etc/network/interfaces`), not netplan. So this play deliberately has **no hostname task and no netplan task**:

    - `ansible.builtin.hostname` needs systemd's dbus, which the DietPi minimal image doesn't run — the task fails with `Failed to connect to system scope bus`.
    - A netplan file would be ignored (netplan isn't installed) and would only fight DietPi's own config.

    Let DietPi own identity; Ansible owns configuration (packages, cgroups, `/etc/hosts`, k3s) — one source of truth per concern. Note the cmdline path is `/boot/firmware/cmdline.txt` on current DietPi / Raspberry Pi OS (Bookworm/Trixie), not the older `/boot/cmdline.txt`.

## Step 7: NFS Playbook

The play is tagged `nfs` so you can bring the cluster up before the SSD is installed: run `ansible-playbook site.yml --skip-tags nfs` now, then `--tags nfs` once the disk is in.

Before running this playbook, format and label the NFS disk on topaz **once**. Identify the disk with `lsblk` (do not assume `/dev/sda` — USB enumeration is not stable across reboots), then:

```bash
ssh dietpi@10.0.20.12 'sudo mkfs.ext4 -L homelab-data /dev/sdX'
```

The mount task below references the filesystem by label, so future reboots resolve it correctly even if the kernel renames the device.

```yaml
- hosts: topaz
  become: true
  tags: [nfs]
  tasks:
    - name: Install NFS server
      ansible.builtin.apt:
        name: nfs-kernel-server
        state: present

    - name: Ensure /data mounted
      ansible.posix.mount:
        path: /data
        src: LABEL=homelab-data
        fstype: ext4
        state: mounted

    - name: Configure NFS export
      ansible.builtin.lineinfile:
        path: /etc/exports
        line: "/data 10.0.20.0/24(rw,sync,no_subtree_check,no_root_squash)"
      notify: reload exports

  handlers:
    - name: reload exports
      ansible.builtin.command: exportfs -ar
```

!!! tip
    `no_root_squash` with subnet restriction is a homelab tradeoff, not a production-grade NFS security posture. It works because every node on `10.0.0.0/24` is yours and the data is yours. Do not copy this config into a multi-tenant environment.

## Step 8: k3s Playbooks

```yaml
- hosts: k3s_server
  become: true
  tasks:
    - name: Install k3s server
      ansible.builtin.shell: |
        curl -sfL https://get.k3s.io | sh -s - \
          --cluster-init \
          --write-kubeconfig-mode 644 \
          --disable servicelb \
          --disable traefik \
          --disable local-storage \
          --disable-cloud-controller \
          --token {{ k3s_token }} \
          --node-ip {{ k3s_server_ip }}
      args:
        creates: /usr/local/bin/k3s

- hosts: k3s_agents
  become: true
  tasks:
    - name: Install k3s agent
      ansible.builtin.shell: |
        curl -sfL https://get.k3s.io | \
          K3S_URL=https://{{ k3s_server_ip }}:6443 \
          K3S_TOKEN={{ k3s_token }} sh -
      args:
        creates: /usr/local/bin/k3s
```

## Step 9: Assembled `site.yml`

The four plays from Steps 6–8 concatenate into one file. Skeleton:

```yaml
---
- name: Bootstrap all nodes (packages, cgroups, /etc/hosts)
  hosts: all
  become: true
  tasks: [...]      # from Step 6

- name: NFS server on topaz
  hosts: topaz
  become: true
  tags: [nfs]
  tasks: [...]      # from Step 7
  handlers: [...]

- name: K3s control plane on ruby
  hosts: k3s_server
  become: true
  tasks: [...]      # from Step 8 (server)

- name: K3s agents on emerald, topaz, amethyst
  hosts: k3s_agents
  become: true
  tasks: [...]      # from Step 8 (agents)
```

Adding a `name:` to each play (which the verbatim Step 6–8 blocks omit for brevity) makes the `ansible-playbook` output much easier to scan.

## Step 10: Secrets via sops + age

The cluster token lives encrypted in `secrets/secrets.sops.yaml` and is decrypted at runtime by the `community.sops` lookup in the inventory — the **same sops + age mechanism** as the `homelab-secrets` repo (Git). That gives you one repo-side secrets tool, consistent with Kubernetes Step 12's two-layer model (sops + age repo-side, Sealed Secrets cluster-side).

Generate a strong token and write it into a sops-encrypted file (the repo-root `.sops.yaml` pins your age recipient):

```bash
sops secrets/secrets.sops.yaml
# Add line:  k3s_token: <output of `openssl rand -hex 32`>
```

The age **private** key stays at `~/.config/sops/age/keys.txt` (never committed); the encrypted file is safe to commit. Save the token to your password manager too — losing it on top of losing ruby turns a 2-hour rebuild into a full cluster rebuild (Kubernetes Step 11).

## Step 11: Run It

```bash
ansible-playbook site.yml --check          # dry run
ansible-playbook site.yml                  # apply
ansible-playbook site.yml --skip-tags nfs  # apply without NFS (SSD not installed yet)
ansible-playbook site.yml --limit emerald  # one node
```

!!! tip
    Commit everything to your `homelab-ansible` repo — including the *encrypted* `secrets/secrets.sops.yaml`. The age private key (`~/.config/sops/age/keys.txt`) lives outside the repo and never gets committed. Combined with ArgoCD watching `homelab-manifests`, you have full infrastructure-as-code: bare-metal provisioning AND application deployment, both from Git.

## Verification

- [ ] Ansible reaches all nodes:

    ```bash
    ansible all -m ping
    # Expected: 4x SUCCESS
    ```

- [ ] Bootstrap idempotent (re-running site.yml shows zero changes):

    ```bash
    ansible-playbook site.yml --check
    # Expected: ok=N changed=0 in summary
    ```

- [ ] `/etc/hosts` populated on every node:

    ```bash
    ansible all -a 'grep -E "ruby|emerald|topaz|amethyst" /etc/hosts'
    ```

- [ ] k3s installed on every node (idempotent):

    ```bash
    ansible all -a 'test -x /usr/local/bin/k3s'
    ```
