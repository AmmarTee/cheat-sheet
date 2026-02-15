# Proxmox CLI Reference

Technical reference for Proxmox VE command-line operations. Compiled during infrastructure shifts and maintenance.

Covers Proxmox 7.x and 8.x.

---

## VM Operations (qm)

### Power Management

**List all VMs:**
```bash
qm list
```

**Start VM:**
```bash
qm start 100  # Replace 100 with your VMID
```

**Stop VM (graceful):**
```bash
qm shutdown 100  # Sends ACPI signal, wait 60-90s if unresponsive
```

**Stop VM (forced):**
```bash
qm stop 100  # ⚠️ Like pulling the power plug, can corrupt data
```

**Reset VM:**
```bash
qm reset 100  # Hard reset, prefer shutdown + start
```

**Check status:**
```bash
qm status 100
```

**⚠️ Destroy VM:**
```bash
qm destroy 100  # Permanently deletes VM and disks, no confirmation
qm destroy 100 --purge 0  # Keeps config for reference
```

### Hardware & Config

**View config:**
```bash
qm config 100
```

**CPU and memory:**
```bash
qm set 100 --cores 4
qm set 100 --memory 4096  # In MB
# Note: Hotplug depends on guest OS
```

**Network:**
```bash
qm set 100 --net0 virtio,bridge=vmbr0  # VirtIO = best performance for Linux
```

**Disks:**
```bash
qm set 100 --scsi1 local-lvm:32  # Add 32GB disk
qm resize 100 scsi0 +20G  # Expand existing disk
# Remember to expand partition inside guest OS after resize
```

**Boot order and ISO:**
```bash
qm set 100 --boot order=scsi0;ide2
qm set 100 --ide2 local:iso/debian-12.iso,media=cdrom
qm set 100 --ide2 none,media=cdrom  # Eject ISO
```

### Cloning & Templates

**Clone VM:**
```bash
qm clone 100 101 --name mc-prod-01  # Full clone
qm clone 100 102 --name plex-srv --full 0  # Linked clone (faster, uses less space)
# Linked clones depend on source VM
```

**Convert to template:**
```bash
qm template 9000  # ⚠️ One-way operation, cannot convert back
```

**Migration:**
```bash
qm migrate 100 pve-node2  # Offline
qm migrate 100 pve-node2 --online  # Live migration (needs shared storage)
```

### Console Access

```bash
qm monitor 100  # QEMU monitor (advanced)
qm terminal 100  # Serial console
# Press Ctrl+O to exit
```

### Creating a VM from Scratch

```bash
# Step 1: Create with basic specs
qm create 100 --name srv-node-01 --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0

# Step 2: Add disk
qm set 100 --scsi0 local-lvm:32

# Step 3: Attach ISO
qm set 100 --ide2 local:iso/ubuntu-22.04-server.iso,media=cdrom

# Step 4: Set boot order (CD first for install)
qm set 100 --boot order=ide2;scsi0

# Step 5: Enable guest agent
qm set 100 --agent enabled=1

# Step 6: Start
qm start 100

# After OS install:
qm set 100 --boot order=scsi0  # Boot from disk
qm set 100 --ide2 none,media=cdrom  # Eject ISO
```

---

## LXC Containers (pct)

### Power Management

**List containers:**
```bash
pct list
```

**Start container:**
```bash
pct start 200  # Replace 200 with your CTID
```

**Stop (graceful):**
```bash
pct shutdown 200  # Sends SIGTERM
```

**Stop (forced):**
```bash
pct stop 200  # ⚠️ Sends SIGKILL immediately
```

**Check status:**
```bash
pct status 200
```

**⚠️ Destroy:**
```bash
pct destroy 200  # Permanently deletes container and rootfs
# Always backup first
```

### Hardware & Config

**View config:**
```bash
pct config 200
```

**Resources:**
```bash
pct set 200 --cores 2
pct set 200 --memory 1024  # In MB
pct set 200 --swap 512
```

**Console access:**
```bash
pct enter 200  # Enter shell
pct exec 200 -- apt update  # Run command without entering
# Type 'exit' to leave
```

**File operations:**
```bash
pct push 200 /root/script.sh /tmp/script.sh  # Copy TO container
pct pull 200 /var/log/app.log /tmp/app.log  # Copy FROM container
```

**Storage:**
```bash
pct resize 200 rootfs 20G  # Total size, not incremental
pct mount 200  # Mount container filesystem
pct unmount 200
```

### Bind Mounts

Share host directories with containers.

```bash
pct set 200 --mp0 /mnt/media,mp=/mnt/media
```

Common uses: NAS storage, persistent data, Docker volumes.

For unprivileged containers, may need UID/GID mapping in `/etc/pve/lxc/200.conf`:
```bash
lxc.idmap: u 0 100000 65536
lxc.idmap: g 0 100000 65536
```

### Template Management

```bash
pveam update  # Update template list
pveam available  # List available
pveam available | grep ubuntu  # Filter by distro
pveam download local ubuntu-22.04-standard_22.04-1_amd64.tar.zst
pveam list local  # List downloaded
```

### Creating a Container

```bash
# Download template if needed
pveam download local ubuntu-22.04-standard_22.04-1_amd64.tar.zst

# Create container
pct create 200 local:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.zst \
  --hostname docker-host \
  --memory 2048 \
  --cores 2 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --storage local-lvm \
  --rootfs local-lvm:8

# Add bind mount (optional)
pct set 200 --mp0 /mnt/docker,mp=/var/lib/docker

# For Docker: Enable privileged mode and nesting
pct set 200 --unprivileged 0
pct set 200 --features nesting=1

# Start
pct start 200

# Install Docker
pct enter 200
apt update && apt install docker.io -y
exit
```

### Privileged vs Unprivileged

**Unprivileged** (more secure, user namespacing):
- Use for: web servers, databases, apps
- Limitation: restricted hardware access

**Privileged** (⚠️ less secure, full host access):
- Use for: Docker, NFS servers, special hardware, FUSE
- Risk: can affect host system

Always use unprivileged unless specifically required.

---

## Storage & Backups (vzdump)

Default location: `/var/lib/vz/dump/`

### Basic Backup

```bash
vzdump 100  # Simple backup
vzdump 100 --storage backup-nfs  # Specify storage
```

### Backup Modes

**snapshot:** No downtime, requires LVM-thin/ZFS/Ceph
```bash
vzdump 100 --mode snapshot
```

**suspend:** Brief downtime (seconds), preserves memory state
```bash
vzdump 100 --mode suspend
```

**stop:** Full downtime, guaranteed consistency, works on any storage
```bash
vzdump 100 --mode stop
```

### Compression

```bash
vzdump 100 --compress zstd  # Best balance (recommended)
vzdump 100 --compress gzip  # Better compatibility
vzdump 100 --compress lzo  # Fastest, less compression
```

### Bulk Backups

```bash
vzdump 100 101 102  # Multiple VMs
vzdump 100 101 102 --mode snapshot --compress zstd
vzdump --all  # ⚠️ Everything (can take hours)
```

### Restore Operations

**List backups:**
```bash
ls -lh /var/lib/vz/dump/
ls -lh /var/lib/vz/dump/vzdump-qemu-100-*
```

**Check backup size:**
```bash
du -sh /var/lib/vz/dump/vzdump-qemu-100-*.vma.zst
du -sh /var/lib/vz/dump/
```

**Restore to same VMID:**
```bash
# ⚠️ VM must not exist or be stopped first
qm stop 100
qm destroy 100  # Optional: backup current state first with vzdump
qmrestore /var/lib/vz/dump/vzdump-qemu-100-*.vma.zst 100
qm start 100
```

**Restore to new VMID (safer):**
```bash
qmrestore /var/lib/vz/dump/vzdump-qemu-100-*.vma.zst 150 --unique
# --unique gives new MAC and machine UUID
```

**Restore to different storage:**
```bash
qmrestore /var/lib/vz/dump/vzdump-qemu-100-*.vma.zst 100 --storage local-lvm
```

**Container restore:**
```bash
pct restore 200 /var/lib/vz/dump/vzdump-lxc-200-*.tar.zst
pct restore 250 /var/lib/vz/dump/vzdump-lxc-200-*.tar.zst --unique  # New CTID
```

### Test Backup Integrity

```bash
# Verify compression (doesn't test functionality)
zstdcat /var/lib/vz/dump/vzdump-qemu-100-*.vma.zst > /dev/null

# Real test: restore and verify
qmrestore backup-file.vma.zst 999 --unique
qm start 999
# Test services, then cleanup
qm destroy 999
```

### Cleanup Old Backups

```bash
find /var/lib/vz/dump/ -name "vzdump-qemu-100-*" -mtime +30  # View old
rm /var/lib/vz/dump/vzdump-qemu-100-2024_01_*.vma.zst  # ⚠️ Delete specific
# Better: use GUI retention settings (Datacenter > Backup)
```

### Disaster Recovery Example

```bash
# Backup
vzdump 100 --mode snapshot --compress zstd --storage local

# Verify
ls -lh /var/lib/vz/dump/vzdump-qemu-100-*

# Simulate disaster
qm stop 100
qm destroy 100

# Restore
qmrestore /var/lib/vz/dump/vzdump-qemu-100-2024_02_15-20_30_00.vma.zst 100
qm start 100

# Verify services
qm status 100
# SSH in and check applications
```

Test your restore process before you need it.

---

## Troubleshooting

### Common Issues

**VM won't start:**
```bash
qm status 100
journalctl -u qemu-server@100
# Check config for invalid disk/network/resource settings
qm config 100
```

**VM stuck starting:**
```bash
journalctl -u pve-cluster
pvesm status
df -h  # Check disk space
# Usually storage unavailable/full, locked config, or network issues
```

**Container permission denied:**
```bash
pct config 200
cat /etc/pve/lxc/200.conf
# If unprivileged, need UID/GID mapping:
# lxc.idmap: u 0 100000 65536
# lxc.idmap: g 0 100000 65536
```

**Out of disk space:**
```bash
df -h
du -sh /var/lib/vz/dump/  # Check backups
# Quick fixes:
rm /var/lib/vz/template/iso/unused-*.iso  # Delete unused ISOs
journalctl --vacuum-time=7d  # Clean logs
```

**VM slow performance:**
```bash
qm status 100 --verbose
qm listsnapshot 100  # ⚠️ Long snapshot chains kill performance
# Fix: Remove old snapshots
qm delsnapshot 100 snapshot-name
```

**Network not working:**
```bash
# Inside VM/CT:
ip a
ping 8.8.8.8

# On host:
brctl show  # Check bridge exists (vmbr0)
ip a
pve-firewall status
```

### Log Locations

**Proxmox system:**
```bash
journalctl -u pve-cluster  # Cluster logs
journalctl -u pvedaemon  # Daemon logs
journalctl -u pve-manager  # Web interface logs
journalctl -xe  # Recent system logs
```

**VM-specific:**
```bash
journalctl -u qemu-server@100.service  # VM start/stop
journalctl -u qemu-server@100.service -n 50  # Last 50 lines
journalctl -u qemu-server@100.service -f  # Follow real-time
```

**Container:**
```bash
pct enter 200
journalctl -n 50
# Or from host:
journalctl CONTAINER_NAME=200
journalctl | grep "CT 200"
```

**Backups:**
```bash
grep vzdump /var/log/syslog
journalctl | grep vzdump
journalctl --since "24 hours ago" | grep vzdump
```

**Task logs (GUI actions):**
```bash
pvesh get /cluster/tasks
cat /var/log/pve/tasks/<task-id>
ls -lht /var/log/pve/tasks/ | head
```

**Web UI:**
```bash
tail -f /var/log/pveproxy/access.log
tail -f /var/log/pveproxy/error.log
```

### Diagnostic Commands

**Cluster & resources:**
```bash
pvesh get /cluster/resources
pvesh get /cluster/resources --output-format=yaml
pvesh get /nodes/pve-node1/status
```

**Storage:**
```bash
pvesm status
pvesm list local
cat /etc/pve/storage.cfg
```

**VM diagnostics:**
```bash
qm config 100
qm showcmd 100  # Show QEMU command
qm listsnapshot 100
qm monitor 100  # Advanced
```

**Container diagnostics:**
```bash
pct config 200
pct exec 200 -- ps aux
pct exec 200 -- df -h
pct exec 200 -- ip a
```

**Network:**
```bash
brctl show  # Bridges
ip -d link show vmbr0
ip a
ip route
pve-firewall status
ping -c 4 8.8.8.8
```

**Storage:**
```bash
df -h
df -i  # Inodes
lvs  # LVM
pvs
vgs
zpool status  # ZFS
zfs list
```

**Performance:**
```bash
htop
iotop  # Install: apt install iotop
nethogs  # Install: apt install nethogs
iostat -x 2
pvesh get /cluster/resources --type vm
free -h
```

---

## Quick Reference

### Most-Used Commands

```bash
# VM operations
qm list
qm start 100
qm shutdown 100
qm config 100
qm set 100 --memory 4096
qm clone 100 101
qm resize 100 scsi0 +20G

# Container operations
pct list
pct start 200
pct enter 200
pct exec 200 -- apt update
pct set 200 --memory 2048

# Backups
vzdump 100 --mode snapshot
qmrestore /var/lib/vz/dump/vzdump-qemu-100-*.vma.zst 150 --unique
pct restore 200 /var/lib/vz/dump/vzdump-lxc-200-*.tar.zst

# Templates
pveam available
pveam download local ubuntu-22.04-standard_22.04-1_amd64.tar.zst

# Storage
pvesm status
du -sh /var/lib/vz/dump/

# Cluster
pvesh get /cluster/resources
```

### Useful One-Liners

**Start/stop all VMs:**
```bash
qm list | awk 'NR>1 {print $1}' | xargs -I {} qm start {}
qm list | awk 'NR>1 {print $1}' | xargs -I {} qm shutdown {}
```

**Start/stop all containers:**
```bash
pct list | awk 'NR>1 {print $1}' | xargs -I {} pct start {}
pct list | awk 'NR>1 {print $1}' | xargs -I {} pct shutdown {}
```

**Reporting:**
```bash
qm list | sort -k4 -rn  # By memory
qm list | grep running | wc -l  # Count running
qm list | awk '$3=="running" {print $1, $2}'  # Only running
```

**Bulk backups:**
```bash
qm list | awk '$3=="running" {print $1}' | xargs -I {} vzdump {} --mode snapshot
qm list | grep "web" | awk '{print $1}' | xargs -I {} vzdump {} --mode snapshot
```

**Find VMs using specific storage:**
```bash
for vm in $(qm list | awk 'NR>1 {print $1}'); do 
  qm config $vm | grep -q "local-lvm" && echo "VM $vm uses local-lvm"
done
```

**Total memory allocated:**
```bash
qm list | awk 'NR>1 {sum+=$4} END {print sum " MB"}'
```

**Clone VM multiple times:**
```bash
for i in {101..105}; do 
  qm clone 100 $i --name "mc-node-$i" --full 1
done
```

**Remove all snapshots from VM:**
```bash
for snap in $(qm listsnapshot 100 | awk 'NR>2 {print $2}'); do
  qm delsnapshot 100 $snap
done
```

**Cleanup:**
```bash
journalctl --vacuum-time=7d  # Clean journal
find /var/lib/vz/dump/ -name "vzdump-*.vma.*" -mtime +30 -ls
# ⚠️ Test with -ls first, then replace with -delete
```

---

## Safety Notes

### Before making changes:

1. Backup: `vzdump 100 --mode snapshot --compress zstd`
2. Save config: `qm config 100 > /tmp/vm-100-config-$(date +%F).txt`
3. Test on non-prod clone
4. Know rollback plan

### ⚠️ Destructive commands:

- `qm destroy` - Deletes VM and disks permanently
- `qm stop` - Hard stop (like power cable pull)
- `qm reset` - Hard reset (can corrupt data)
- `pct destroy` - Deletes container permanently
- `rm` on backups - Gone forever

### Power options (safest to most aggressive):

1. **Graceful:** `qm shutdown 100` (ACPI signal, wait 60-90s)
2. **Forced:** `qm shutdown 100 --timeout 30 --forceStop 1` (tries graceful first)
3. **Hard:** `qm stop 100` (⚠️ immediate power off, last resort)

### Testing backups:

Monthly minimum, quarterly for non-critical:
```bash
qmrestore /var/lib/vz/dump/vzdump-qemu-100-latest.vma.zst 999 --unique
qm start 999
# SSH in, verify services, check data integrity
qm destroy 999
echo "$(date): Backup test successful for VM 100" >> /root/backup-tests.log
```

### Resource management:

- Proxmox needs 10-20% free space
- Leave 25% RAM for host
- CPU overcommit 2:1 is safe
- Monitor with alerts

### Common mistakes:

- MAC conflicts on clones (use `--unique`)
- Bridge doesn't exist (verify `brctl show` first)
- Firewall blocks traffic (`pve-firewall status`)
- No network in cloned VMs (use `--unique`)

---

## Resources

**Official docs:**
- https://pve.proxmox.com/pve-docs/pve-admin-guide.html
- https://pve.proxmox.com/pve-docs/api-viewer/
- https://pve.proxmox.com/wiki/Main_Page
- https://forum.proxmox.com/

**Video tutorials:**
- LearnLinuxTV Proxmox playlist
- TechnoTim guides
- Craft Computing

**Communities:**
- r/Proxmox
- r/homelab
- r/selfhosted

**Automation:**
- Terraform Proxmox Provider: https://registry.terraform.io/providers/Telmate/proxmox/latest/docs
- Ansible Proxmox Modules: https://docs.ansible.com/ansible/latest/collections/community/general/proxmox_module.html
- Proxmox Helper Scripts: https://community-scripts.github.io/Proxmox/

---

Last updated: Feb 2026 | Proxmox 7.x & 8.x
