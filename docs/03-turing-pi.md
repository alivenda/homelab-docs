# Runbook 3: Turing Pi 2 Board Setup

Initial hardware assembly, firmware update, OS flashing, and network configuration for your Turing Pi 2 with 4× Raspberry Pi CM4 modules (mixed eMMC capacity).

## Hardware Inventory

| | |
|---|---|
| **Board** | Turing Pi 2 (mini-ITX cluster board) |
| **Modules** | 4× CM4, all with 8 GB RAM and WiFi |
| **High-storage nodes** | 2× CM4108032 (8 GB RAM, 32 GB eMMC) — ruby (Node 1), emerald (Node 2) |
| **Low-storage nodes** | 2× CM4108016 (8 GB RAM, 16 GB eMMC) — topaz (Node 3), amethyst (Node 4) |
| **Total Cluster** | 16 cores, 32 GB RAM, 96 GB eMMC combined |
| **Networking** | Onboard 1 Gbps managed switch, 2× RJ45 (bridged) |
| **Storage I/O** | 2× SATA III (Node 3), 4× M.2 slots (NOT usable with CM4) |
| **Time Estimate** | 1–2 hours (or 15 min with Runbook 4 / Ansible) |

!!! warning "M.2 slots and CM4"
    The 4 M.2 slots on the back of the board are NOT usable with Raspberry Pi CM4 modules. Per the official Turing Pi docs, CM4 lacks the PCIe lanes needed to reach them. Only RK1 and Jetson modules can use the M.2 slots. Use a single SATA SSD on Node 3 for cluster persistent storage.

## Node Assignment Strategy

| Node | Hardware | Role |
|---|---|---|
| `ruby (Node 1)` (10.0.0.60) | 32 GB eMMC | k3s control plane + worker; image cache |
| `emerald (Node 2)` (10.0.0.61) | 32 GB eMMC | Worker for heavier apps (databases, Forgejo, Nextcloud) |
| `topaz (Node 3)` (10.0.0.62) | 16 GB eMMC + SATA SSD | NFS server + light worker (data on SSD) |
| `amethyst (Node 4)` (10.0.0.63) | 16 GB eMMC | Light worker (Vaultwarden, monitoring agents) |

## Step 1: Hardware Assembly

1. Install CM4 modules onto the Turing Pi 2 adapter boards. Align the sides and notch with the slot, then press vertically until the side arms click into place.
2. Place the 32 GB modules in slots 1 and 2 (ruby (Node 1), emerald (Node 2)). Place the 16 GB modules in slots 3 and 4 (topaz (Node 3), amethyst (Node 4)).
3. Attach heatsinks to each CM4 (Waveshare aluminum heatsinks with thermal tape work well).
4. Insert the adapter boards into the 4 SO-DIMM slots on the Turing Pi 2 board.
5. Connect a SATA SSD to the onboard SATA connector for topaz (Node 3) (Node 3 slot). Do **not** hot-plug — connect while powered off.
6. Connect one RJ45 port to your UDM/network switch.
7. Connect a PicoPSU or ATX PSU via the 24-pin ATX connector. Power up.

## Step 2: Access the BMC

The BMC (Baseboard Management Controller) starts automatically within 10–20 seconds.

1. Open a browser to `turingpi.local` (or find the IP via your UDM).
2. Verify SSH access: `ssh root@turingpi.local`
3. Update the BMC firmware to the latest version. See [docs.turingpi.com](https://docs.turingpi.com) for the firmware upgrade guide.

## Step 3: Flash DietPi OS to All 4 Modules

DietPi is a lightweight Debian-based OS optimized for single-board computers and recommended in the official Turing Pi k3s guide.

1. Download the DietPi image for Raspberry Pi 2/3/4 (ARM 64-bit) from [dietpi.com](https://dietpi.com).
2. Install `rpiboot` on your PC (required for the PC to see CM4 eMMC storage). Windows: [rpiboot_setup.exe](https://github.com/raspberrypi/usbboot/raw/master/win32/rpiboot_setup.exe). Mac/Linux: build from [raspberrypi/usbboot](https://github.com/raspberrypi/usbboot).
3. Connect the vertical USB 2.0 port on the Turing Pi 2 (adjacent to HDMI) to your PC with a **USB A-to-A cable**. The Turing Pi docs explicitly note that USB A-to-C cables have caused intermittent failures — use A-to-A.
4. Power on the target node (BMC UI or `tpi power on -n <node>`) before the next step — MSD mode requires the module to be running.
5. For each node, put it into USB mass-storage mode so your PC sees the eMMC as a disk. SSH into the BMC and run:

    ```bash
    # Route node 1's eMMC to the BMC USB port as a mass-storage device
    tpi usb device -n 1
    ```

    Or use the BMC web UI: **Nodes → select node → USB → Device mode**. On your PC, `rpiboot` should detect the CM4 storage and present it as a new disk.

6. Flash via Raspberry Pi Imager using "Use custom" with the DietPi image.
7. Before completing the flash, mount the boot partition and edit `dietpi.txt`:

    ```ini
    AUTO_SETUP_LOCALE=C.UTF-8
    AUTO_SETUP_KEYBOARD_LAYOUT=us
    AUTO_SETUP_TIMEZONE=America/New_York
    AUTO_SETUP_NET_ETHERNET_ENABLED=1
    AUTO_SETUP_NET_WIFI_ENABLED=0
    AUTO_SETUP_NET_USESTATIC=1
    AUTO_SETUP_NET_STATIC_IP=10.0.0.60     # Increment per node: .60, .61, .62, .63
    AUTO_SETUP_NET_STATIC_MASK=255.255.255.0
    AUTO_SETUP_NET_STATIC_GATEWAY=10.0.0.1
    AUTO_SETUP_NET_STATIC_DNS=1.1.1.1 8.8.8.8
    AUTO_SETUP_NET_HOSTNAME=ruby           # ruby (Node 1), emerald (Node 2), topaz (Node 3), amethyst (Node 4)
    AUTO_SETUP_HEADLESS=1
    AUTO_SETUP_AUTOSTART_TARGET_INDEX=7
    AUTO_SETUP_AUTOSTART_LOGIN_USER=root
    AUTO_SETUP_GLOBAL_PASSWORD=<YOUR_PASSWORD>
    AUTO_SETUP_SSH_SERVER_INDEX=-2
    ```

8. Also edit `cmdline.txt` to enable cgroups (required for k3s). Append to the end of the single line: `cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1`
9. Repeat for all 4 nodes, incrementing the static IP and hostname.

!!! tip "Jump to Ansible now"
    **STOP HERE and jump to Runbook 4 (Ansible).** Steps 4 and 5 below are MANUAL alternatives — Runbook 4 automates the cgroups edit, NFS server install, SSD mount, and k3s install across all 4 nodes with one playbook. Only do Steps 4–5 manually if you want to understand the per-node config before letting Ansible take over.

## Step 4: Boot and Verify

1. Power on all nodes via the BMC UI or by pressing the `Key1` button.
2. Wait for first boot (2–3 minutes).
3. SSH into each node and update:

    ```bash
    ssh root@10.0.0.60
    apt update && apt upgrade -y
    ```

## Step 5: Prepare SATA SSD on Node 3 (topaz (Node 3))

!!! note "Superseded by Ansible"
    Runbook 4's NFS playbook does this automatically. This step is here as the imperative reference for what Ansible runs.

Node 3 connects to the SATA ports. This SSD will serve as NFS storage for the entire cluster.

1. SSH into topaz (Node 3) and identify the disk: `fdisk -l`
2. Format and mount:

    ```bash
    mkfs.ext4 /dev/sda
    mkdir /data
    echo "/dev/sda /data ext4 defaults 0 0" | tee -a /etc/fstab
    mount -a
    df -h /data
    ```

!!! tip
    Reserve IPs 10.0.0.60–63 for your nodes and 10.0.0.70–80 for MetalLB in your UDM DHCP settings.

## Power: UPS + NUT for Graceful Shutdown

A 4-node CM4 cluster plus a NAS draws modest power, but an unscheduled power loss can corrupt etcd, leave NFS exports in inconsistent state, and force fsck on every node next boot. A small UPS pays for itself the first time you have a brownout during a cluster upgrade.

**Sizing rule of thumb:** a Turing Pi 2 with 4 CM4s + small SSD pulls 30–60 W under load. A 600–900 VA UPS gives 15–30 minutes of runtime — plenty for graceful shutdown of the cluster plus NAS. Common picks: APC Back-UPS BX700U, CyberPower CP900AVR.

What you actually need from the UPS is not the runtime, it is the USB or network signal that the UPS sends when on battery. [Network UPS Tools (NUT)](https://networkupstools.org/) is the open-source daemon that listens for that signal and triggers shutdown scripts across multiple hosts.

**Topology:**

- UPS connects to one host (typically the NAS, since it runs 24/7) via USB. That host runs `nut-server` in `upsd` mode and exposes the UPS status over the network.
- Every cluster node runs `nut-client` in `upsmon` mode, talking to the NAS `upsd`. When `upsd` reports `LOW_BATTERY`, all clients run their configured shutdown command.
- Order matters: workers shut down first, then control plane, then NAS last (so NFS stays mounted until the cluster is fully drained).

```bash
# On NAS (UPS master)
sudo apt install nut
# Configure /etc/nut/ups.conf with your UPS model
# Set MODE=netserver in /etc/nut/nut.conf

# On each k3s node (slaves)
sudo apt install nut-client
# Configure /etc/nut/upsmon.conf to point at NAS
# MONITOR ups@<nas-ip> 1 monuser <password> slave
```

!!! tip
    Test the shutdown flow by unplugging the UPS from the wall (not the equipment from the UPS). You want to see the cascade actually happen before a real outage proves your config wrong.

This is a maturity layer — skip it for the first cluster build, add it once everything else is working. But add it before you store anything you care about losing.

## Verification

- [ ] All 4 nodes reachable via SSH:

    ```bash
    for i in 60 61 62 63; do
      ssh -o ConnectTimeout=3 root@10.0.0.$i 'hostname && uname -m'
    done
    # Expected: ruby / emerald / topaz / amethyst each printing aarch64
    ```

- [ ] cgroups enabled on each node (required for k3s):

    ```bash
    ssh root@10.0.0.60 grep -o 'cgroup_enable=[a-z]*\|cgroup_memory=1' /boot/cmdline.txt
    ```

- [ ] On topaz (Node 3) specifically, the SATA SSD is mounted at `/data`:

    ```bash
    ssh root@10.0.0.62 'df -h /data && mount | grep /data'
    ```
