# sea-pegasus-ansible

Ansible for **pegasus** (`192.168.0.253`), a Debian 13 (trixie) KVM/libvirt
host that runs Pterodactyl game-server VMs. Manages ZFS storage, VLAN-50
`systemd-networkd` bridging, cloud-init VM provisioning, and the Pterodactyl
Panel + Wings stack inside guest VMs.

See [CLAUDE.md](CLAUDE.md) for the authoritative, in-depth reference.

## Status

Pegasus is **powered off since ~2026-05** to save power while no game servers
are in use. Its Prometheus scrape targets in `sea-k8s-flux` are commented out,
and the weekly Semaphore apt-upgrade against it fails with `No route to host`
(expected). Bringing it back requires physical power-on / WoL — nothing here
automates that.

## Host

- **pegasus**, Debian 13 trixie, SSH `pi@192.168.0.253`
- **Storage**: ZFS **RAIDZ1** pool `vmpool` on 4x SATA disks (`/dev/sd[a-d]`),
  mounted at `/vmpool` and used as the libvirt storage pool path.
- **Network**: single trunk port `enp4s0` carrying VLAN 1 (untagged mgmt) +
  VLAN 50 (services/VMs), bridged as `br0` + `br-vlan50`.

## Inventory & secrets

`inventory.yml` **is committed to git** (there is no sample file). It holds
**no secrets** — `nut_client_password`, `pterodactyl_panel_db_password`,
`pterodactyl_panel_app_key`, `pterodactyl_panel_hashids_salt`, and
`mysql_root_password` are injected at runtime via the Semaphore "Default"
environment (ID 4). Production runs go through Semaphore project
**"Sea Pegasus VM Host"** (ID 2).

## Usage

```bash
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install -r requirements.yml

ansible vm_hosts -m ping                       # connectivity
ansible-playbook playbooks/site.yml            # full host provisioning
ansible-playbook playbooks/check-system.yml    # health check
```

The `network` role is `never`-tagged in `site.yml`; apply it explicitly with
`playbooks/network-only.yml` **only with console access** (the
`systemd-networkd` cutover can drop SSH). `playbooks/network-rollback.yml`
reverts to ifupdown in an emergency.

Other key playbooks: `pterodactyl.yml` / `node-2.yml` (create VMs),
`pterodactyl-backup.yml` (DB + `.env` backup), `pterodactyl-update.yml`
(Panel + Wings update), `apt-upgrade.yml` (host, notify-only),
`apt-upgrade-vms.yml` (VMs, serial reboot), `nut-client.yml` (UPS shutdown).

## Roles

Local: `prereq`, `zfs`, `network`, `libvirt`, `cockpit`, `create-vm`,
`nut-shutdown`, `pterodactyl-apache-proxy`. Shared:
`reenchree.common.{node_exporter,zfs_exporter,smartctl_exporter}`. External:
`maxhoesel.pterodactyl` (Panel + Wings), `geerlingguy.nut_client`.
