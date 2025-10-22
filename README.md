# sea-vms-ansible

Ansible project to configure a Debian-based VM host with libvirt/KVM, ZFS storage, and VLAN networking.

## Overview

This playbook sets up a complete VM host environment on Debian with:
- **ZFS** mirror pool on 2x Samsung EVO 500GB drives
- **libvirt/KVM** for virtualization
- **Cockpit** web UI with VM management
- **systemd-networkd** with VLAN 50 bridge for VM networking

## Prerequisites

- Debian 13 (trixie) installed on target host
- SSH access with passwordless sudo
- Two identical drives for ZFS pool
- Network interface configured as trunk port (carrying all VLANs)

## Initial Setup

1. Copy the sample inventory:
   ```bash
   cp inventory-sample.yml inventory.yml
   ```

2. Edit `inventory.yml` with your specific configuration:
   - Host IP address
   - ZFS device paths
   - Network interface name
   - VLAN ID

3. Test ansible connectivity:
   ```bash
   ansible vm_hosts -m ping
   ```

## Deployment Steps

### Step 1: Prerequisites and ZFS

Run everything except networking:

```bash
ansible-playbook playbooks/site.yml --skip-tags network
```

This installs:
- System packages
- ZFS and creates the mirror pool
- libvirt/KVM
- Cockpit web UI

Verify with:
```bash
ansible-playbook playbooks/check-system.yml
```

### Step 2: Network Configuration (DANGEROUS!)

**⚠️ WARNING**: Network changes can break SSH connectivity!

The network role switches from ifupdown to systemd-networkd and creates:
- `br0` - Bridge on untagged VLAN (host management, DHCP)
- `br-vlan50` - Bridge on VLAN 50 (for VMs)

**Before running:**
- Ensure you have physical/console access to the host
- Verify your switch port is configured as trunk with native VLAN 1

**To apply network changes:**

```bash
ansible-playbook playbooks/network-only.yml
```

This playbook will:
1. Backup current network config
2. Show a warning and wait for confirmation
3. Apply systemd-networkd configuration
4. Restart networking
5. Test connectivity

**If something goes wrong:**

You have three options:

1. **Via console/physical access**: Manually revert `/etc/network/interfaces`
2. **If SSH still works**: Run the rollback playbook
   ```bash
   ansible-playbook playbooks/network-rollback.yml
   ```
3. **Check IP address changed**: Update inventory.yml and retry

### Step 3: Verify Everything

```bash
ansible-playbook playbooks/check-system.yml
```

Check:
- ZFS pool is healthy
- Libvirt storage pool exists
- Bridges `br0` and `br-vlan50` are up
- VLAN interface exists

## Accessing the System

- **Cockpit Web UI**: `https://192.168.0.253:9090`
- **SSH**: `ssh pi@192.168.0.253`
- **VMs**: Connect to `br-vlan50` bridge to get VLAN 50 networking

## VM Networking

When creating VMs:
- **Network**: Use bridge `br-vlan50`
- **DHCP**: VMs will get IPs from router on 192.168.50.0/24
- **VLAN**: Traffic is tagged with VLAN 50

Optional: VMs can also use `br0` for untagged networking.

## Common Operations

### Check ZFS pool status
```bash
ssh 192.168.0.253 "zpool status vmpool"
```

### List VMs
```bash
ssh 192.168.0.253 "virsh list --all"
```

### Check network configuration
```bash
ssh 192.168.0.253 "networkctl status"
```

### Check bridges
```bash
ssh 192.168.0.253 "ip -br link show type bridge"
```

## Project Structure

```
.
├── ansible.cfg           # Ansible configuration
├── inventory.yml         # Host inventory (gitignored)
├── inventory-sample.yml  # Sample inventory
├── playbooks/
│   ├── site.yml          # Main playbook
│   ├── network-only.yml  # Network changes only
│   ├── network-rollback.yml # Emergency rollback
│   └── check-system.yml  # System status check
└── roles/
    ├── prereq/           # System prerequisites
    ├── zfs/              # ZFS pool setup
    ├── network/          # systemd-networkd config
    ├── libvirt/          # KVM/libvirt setup
    └── cockpit/          # Cockpit web UI
```

## Safety Notes

- **Secrets**: `inventory.yml` is gitignored to prevent committing credentials
- **ZFS**: Playbook won't recreate pool if it already exists
- **Network**: Use `network-only.yml` playbook separately first
- **Idempotent**: Safe to re-run playbooks multiple times

## Troubleshooting

### Lost SSH after network change
- Access via console/physical connection
- Check `/etc/systemd/network/` configs
- Revert to backup: `cp /etc/network/interfaces.backup /etc/network/interfaces`
- Enable ifupdown: `systemctl enable networking && systemctl disable systemd-networkd`
- Reboot

### ZFS pool issues
- Check status: `zpool status`
- Check devices: `lsblk`
- Import manually: `zpool import vmpool`

### Libvirt issues
- Check service: `systemctl status libvirtd`
- Check pools: `virsh pool-list --all`
- Check networks: `virsh net-list --all`

## License

MIT
