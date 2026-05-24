# Runbook 4: Ansible Automation

Replace the manual per-node steps in Runbook 3 with a single Ansible playbook. Provision all 4 CM4 nodes (or rebuild any one) in minutes.

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Time Estimate** | 2–3 hours upfront, saves hours forever |
| **Runs On** | Your machine (the Ansible control host) |
| **Replaces** | Manual setup steps in Runbook 3 + k3s install |
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

!!! tip
    Add the `kubernetes.core` collection to your `requirements.yml` before R5: `ansible-galaxy collection install kubernetes.core`. The `kubeconfig:` parameter on `kubernetes.core.k8s` lets you run cluster ops from your machine against the remote API.

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

```bash
ssh-keygen -t ed25519 -C "ansible@homelab"   # If you don't have one
for i in 60 61 62 63; do
  ssh-copy-id root@10.0.0.$i
done
```

## Step 3: Project Structure

```
homelab-ansible/
├── inventory.yml
├── ansible.cfg
├── playbooks/
│   ├── site.yml
│   ├── bootstrap.yml
│   ├── k3s-server.yml
│   ├── k3s-agents.yml
│   └── nfs-server.yml
└── roles/
    └── common/
        └── tasks/
            └── main.yml
```

## Step 4: Inventory (`inventory.yml`)

```yaml
all:
  hosts:
    ruby:
      ansible_host: 10.0.0.60
      node_role: server
      emmc_size: 32
    emerald:
      ansible_host: 10.0.0.61
      node_role: agent
      emmc_size: 32
    topaz:
      ansible_host: 10.0.0.62
      node_role: agent
      emmc_size: 16
      nfs_server: true
    amethyst:
      ansible_host: 10.0.0.63
      node_role: agent
      emmc_size: 16
  vars:
    ansible_user: root
    ansible_python_interpreter: /usr/bin/python3
    k3s_token: "{{ vault_k3s_token }}"
    k3s_server_ip: 10.0.0.60

k3s_server:
  hosts:
    ruby:

k3s_agents:
  hosts:
    emerald:
    topaz:
    amethyst:
```

!!! tip
    The inventory references `{{ vault_k3s_token }}`. We create that variable in Step 10 (Secrets via Ansible Vault). Ansible only resolves Jinja at task-execution time, so as long as you complete Step 10 before running `ansible-playbook` in Step 11, this works. Just don't run any playbook before Step 10 or Ansible will error with `vault_k3s_token is undefined`.

## Step 5: `ansible.cfg`

```ini
[defaults]
inventory = inventory.yml
host_key_checking = True
retry_files_enabled = False
stdout_callback = yaml
vault_password_file = ~/.ansible-vault-pass
```

## Step 6: Bootstrap Playbook

```yaml
- hosts: all
  become: true
  tasks:
    - name: Set hostname
      ansible.builtin.hostname:
        name: "{{ inventory_hostname }}"

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
        path: /boot/cmdline.txt
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
          10.0.0.60 ruby
          10.0.0.61 emerald
          10.0.0.62 topaz
          10.0.0.63 amethyst
```

## Step 7: NFS Playbook

```yaml
- hosts: topaz
  become: true
  tasks:
    - name: Install NFS server
      ansible.builtin.apt:
        name: nfs-kernel-server
        state: present

    - name: Ensure /data mounted
      ansible.posix.mount:
        path: /data
        src: /dev/sda
        fstype: ext4
        state: mounted

    - name: Configure NFS export
      ansible.builtin.lineinfile:
        path: /etc/exports
        line: "/data 10.0.0.0/24(rw,sync,no_subtree_check,no_root_squash)"
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
          --write-kubeconfig-mode 644 \
          --disable servicelb \
          --disable traefik \
          --disable local-storage \
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

## Step 9: Site Playbook (Top-Level)

```yaml
- import_playbook: bootstrap.yml
- import_playbook: nfs-server.yml
- import_playbook: k3s-server.yml
- import_playbook: k3s-agents.yml
```

## Step 10: Secrets via Ansible Vault

```bash
ansible-vault create group_vars/all/vault.yml
# Add line: vault_k3s_token: <STRONG_RANDOM_TOKEN>
```

Generate a strong token with `openssl rand -hex 32` and save to Vaultwarden. Save the vault password to `~/.ansible-vault-pass`.

## Step 11: Run It

```bash
ansible-playbook playbooks/site.yml --check    # dry run
ansible-playbook playbooks/site.yml            # apply
ansible-playbook playbooks/bootstrap.yml --limit emerald
```

!!! tip
    Commit this entire `ansible/` directory to your `homelab-ansible` repo. Combined with ArgoCD watching `homelab-manifests`, you have full infrastructure-as-code: bare-metal provisioning AND application deployment, both from Git.

## Verification

- [ ] Ansible reaches all nodes:

    ```bash
    ansible all -m ping
    # Expected: 4x SUCCESS
    ```

- [ ] Bootstrap idempotent (re-running site.yml shows zero changes):

    ```bash
    ansible-playbook playbooks/site.yml --check
    # Expected: ok=N changed=0 in summary
    ```

- [ ] `/etc/hosts` populated on every node:

    ```bash
    ansible all -a 'grep cube /etc/hosts'
    ```

- [ ] k3s installed on every node (idempotent):

    ```bash
    ansible all -a 'test -x /usr/local/bin/k3s'
    ```
