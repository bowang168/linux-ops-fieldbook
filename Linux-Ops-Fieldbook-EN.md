# Linux Ops Fieldbook

## Production Troubleshooting & Performance Guide

---

**Author: Bo Wang**
📧 v0279158888@gmail.com | [GitHub](https://github.com/bowang168) | [LinkedIn](https://www.linkedin.com/in/bowang168) | [Blog](https://bowang168.github.io)
**Version: v1.0 | 2026 Edition**

---

> *"In production, you don't need to know everything — you need to know where to look in 5 minutes."*

This book is written for experienced Linux engineers, SREs, DevOps practitioners, and technical support professionals. It covers the full stack — from system boot to container orchestration, from disk troubleshooting to performance tuning. All commands and configurations are based on **RHEL 8/9 and compatible distributions (Rocky, Alma, Oracle Linux)** and are relevant for production environments from 2024 through 2030.

---

## 📖 About the Author

**Bo Wang** is a Principal Technical Support Engineer at Oracle New Zealand with 17 years of enterprise IT experience across Oracle and IBM. He holds RHCE, CCNP, PMP, and OCI certifications. His expertise spans Linux kernel tuning, cloud infrastructure (OCI, AWS), storage architecture, network performance optimization, and large-scale incident response across 1,200+ production environments. This book distills years of frontline troubleshooting notes and best practices into a structured, actionable reference.

- GitHub: [github.com/bowang168](https://github.com/bowang168)
- LinkedIn: [linkedin.com/in/bowang168](https://www.linkedin.com/in/bowang168)
- Blog: [bowang168.github.io](https://bowang168.github.io)

---

## Table of Contents

- [Chapter 1: Boot Process & Recovery](#chapter-1-boot-process--recovery)
- [Chapter 2: Kernel Management & Tuning](#chapter-2-kernel-management--tuning)
- [Chapter 3: Storage & Filesystems](#chapter-3-storage--filesystems)
- [Chapter 4: Networking & Diagnostics](#chapter-4-networking--diagnostics)
- [Chapter 5: Security & Compliance](#chapter-5-security--compliance)
- [Chapter 6: Core Services](#chapter-6-core-services)
- [Chapter 7: Containers & Cloud Native](#chapter-7-containers--cloud-native)
- [Chapter 8: Shell & Automation](#chapter-8-shell--automation)
- [Chapter 9: Performance Analysis & Tuning](#chapter-9-performance-analysis--tuning)
- [Chapter 10: Diagnostic Tools & Methodology](#chapter-10-diagnostic-tools--methodology)
- [Appendix A: Quick Reference Sheets](#appendix-a-quick-reference-sheets)

---

# Chapter 1: Boot Process & Recovery

> 🎯 **Goal**: Understand the Linux boot sequence end-to-end and recover from GRUB corruption, kernel panics, and lost root passwords.

## 1.1 Boot Sequence Overview

```
Firmware (UEFI/BIOS) → GRUB2 → Kernel → initramfs → systemd → Multi-user target
```

| Stage | Component | Key Files |
|-------|-----------|-----------|
| Firmware | UEFI / Legacy BIOS | NVRAM / MBR |
| Bootloader | GRUB2 | `/boot/grub2/grub.cfg` |
| Kernel | vmlinuz | `/boot/vmlinuz-*` |
| Init filesystem | initramfs | `/boot/initramfs-*.img` |
| Init system | systemd (PID 1) | `/etc/systemd/system/` |

## 1.2 GRUB2 Management

```bash
# View current default
grub2-editenv list

# List available kernels
grubby --info=ALL | grep ^kernel

# Set default kernel
grub2-set-default 0
grubby --set-default /boot/vmlinuz-5.14.0-427.el9.x86_64

# Regenerate config
grub2-mkconfig -o /boot/grub2/grub.cfg        # BIOS
grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg  # UEFI

# Password protection
grub2-setpassword
```

## 1.3 Recovery Quick Reference

| Scenario | Recovery Steps |
|----------|---------------|
| **Lost root password** | GRUB edit → add `rd.break` → `chroot /sysroot` → `passwd root` → `touch /.autorelabel` |
| **Kernel panic** | Select previous kernel in GRUB, fix and set default |
| **Corrupt initramfs** | Rescue mode → `dracut -f /boot/initramfs-$(uname -r).img $(uname -r)` |
| **GRUB overwritten** | Rescue media → `chroot /mnt/sysimage` → `grub2-install /dev/sda` |
| **UEFI entry missing** | `efibootmgr -c -d /dev/sda -p 1 -L "Linux" -l "\EFI\redhat\shimx64.efi"` |
| **Emergency mode** | Check `/etc/fstab` entries → `systemctl default` |

## 1.4 initramfs Management

```bash
# Rebuild for current kernel
dracut -f

# Rebuild all kernels
dracut -f --regenerate-all

# Host-only (smaller image)
dracut --hostonly -f

# Inspect contents
lsinitrd /boot/initramfs-$(uname -r).img | head -50

# Add extra drivers
dracut --add-drivers "mpt3sas megaraid_sas" -f
```

## 1.5 systemd Boot Targets

```bash
# View/change default target
systemctl get-default
systemctl set-default multi-user.target

# Boot time analysis
systemd-analyze
systemd-analyze blame | head -20
systemd-analyze critical-chain
```

---

# Chapter 2: Kernel Management & Tuning

> 🎯 **Goal**: Manage kernel versions, modules, sysctl parameters, and crash dumps.

## 2.1 Kernel Version Management

```bash
uname -r
rpm -qa kernel-core | sort
dnf install kernel
grubby --set-default /boot/vmlinuz-<version>

# Remove old kernels (keep latest 2)
dnf remove --oldinstallonly --setopt installonly_limit=2 kernel
```

## 2.2 Kernel Module Management

```bash
lsmod                            # List loaded modules
modinfo e1000e                   # Module details
modprobe bonding                 # Load
modprobe -r bonding              # Unload

# Persistent loading
echo "bonding" > /etc/modules-load.d/bonding.conf

# Blacklisting
cat > /etc/modprobe.d/blacklist-nouveau.conf << EOF
blacklist nouveau
options nouveau modeset=0
EOF
dracut -f                        # Rebuild initramfs
```

## 2.3 sysctl Kernel Parameters

```bash
cat > /etc/sysctl.d/99-tuning.conf << EOF
# Memory
vm.swappiness = 10
vm.dirty_ratio = 15
vm.dirty_background_ratio = 5

# Network
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.ip_local_port_range = 1024 65535

# File handles
fs.file-max = 2097152
fs.inotify.max_user_watches = 524288
EOF

sysctl -p /etc/sysctl.d/99-tuning.conf
```

### Commonly Tuned Parameters

| Parameter | Default | Recommended | Purpose |
|-----------|---------|-------------|---------|
| `vm.swappiness` | 60 | 10 | Reduce swap tendency (1 for databases) |
| `vm.dirty_ratio` | 20 | 15 | Max dirty page ratio before forced write |
| `net.core.somaxconn` | 4096 | 65535 | Max TCP listen backlog |
| `net.ipv4.tcp_tw_reuse` | 2 | 1 | Allow TIME_WAIT socket reuse |
| `fs.file-max` | varies | 2097152 | System-wide max file descriptors |

## 2.4 Transparent Huge Pages (THP)

```bash
# Check status
cat /sys/kernel/mm/transparent_hugepage/enabled

# Disable (recommended for databases)
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# Persistent: add to GRUB kernel cmdline
# transparent_hugepage=never

# Static Huge Pages (Oracle DB, SAP HANA)
sysctl -w vm.nr_hugepages=1024
grep Huge /proc/meminfo
```

## 2.5 UDEV Rules

```bash
# Inspect device attributes
udevadm info -a -n /dev/sdb

# Custom rule example
cat > /etc/udev/rules.d/99-custom.rules << EOF
ACTION=="add|change", KERNEL=="sd*", ENV{ID_SERIAL}=="3600...", SYMLINK+="oracleasm/disk1"
EOF

udevadm control --reload-rules && udevadm trigger
```

## 2.6 Kdump Configuration

```bash
dnf install kexec-tools
systemctl enable --now kdump
kdumpctl status

# Config: /etc/kdump.conf
# path /var/crash
# core_collector makedumpfile -l --message-level 1 -d 31

# ⚠️ Test (triggers immediate crash + reboot)
echo c > /proc/sysrq-trigger

# Analyze vmcore
crash /usr/lib/debug/lib/modules/$(uname -r)/vmlinux \
      /var/crash/<timestamp>/vmcore
```

## 2.7 Magic SysRq (Emergency Recovery)

Safe reboot sequence **R-E-I-S-U-B** ("Reboot Even If System Utterly Broken"):

| Key | Action |
|-----|--------|
| **R** | Take keyboard back from X |
| **E** | Send SIGTERM to all processes |
| **I** | Send SIGKILL to all processes |
| **S** | Sync all filesystems |
| **U** | Remount all filesystems read-only |
| **B** | Reboot immediately |

```bash
echo 1 > /proc/sys/kernel/sysrq   # Enable
echo s > /proc/sysrq-trigger      # Sync
echo u > /proc/sysrq-trigger      # Remount ro
echo b > /proc/sysrq-trigger      # Reboot
```

---

# Chapter 3: Storage & Filesystems

> 🎯 **Goal**: Master disk partitioning, LVM, multipath, NVMe, RAID, iSCSI, encryption, and filesystem operations.

## 3.1 Block Devices & Partitioning

| Tool | Partition Table | Max Size | Use Case |
|------|----------------|----------|----------|
| `fdisk` | MBR | 2 TiB | Legacy BIOS |
| `gdisk` | GPT | 8 ZiB | UEFI, large disks |
| `parted` | MBR + GPT | Unlimited | Scripted partitioning |

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,MODEL
parted -s -a optimal /dev/sdb mklabel gpt mkpart primary 0% 100%
partprobe /dev/sdb
blkid /dev/sdb1
```

## 3.2 LVM (Logical Volume Manager)

```
Physical Disk → PV → VG → LV → Filesystem
```

```bash
# Create
pvcreate /dev/sdb1 /dev/sdc1
vgcreate vg_data /dev/sdb1 /dev/sdc1
lvcreate -L 200G -n lv_app vg_data
mkfs.xfs /dev/vg_data/lv_app
mount /dev/vg_data/lv_app /app

# Extend (online)
vgextend vg_data /dev/sdd1
lvextend -L +100G /dev/vg_data/lv_app
xfs_growfs /app        # XFS
resize2fs /dev/vg_data/lv_app   # Ext4

# Snapshot
lvcreate -L 1G -s -n snap /dev/vg_data/lv_app

# Disaster recovery
vgcfgbackup vg_data
vgcfgrestore -f /etc/lvm/backup/vg_data vg_data
vgchange -ay vg_data

# Status
pvs && vgs && lvs
```

## 3.3 Filesystems

| Feature | XFS | Ext4 | Btrfs |
|---------|-----|------|-------|
| RHEL Default | 8/9 | 6/7 | Experimental |
| Max Volume | 8 EiB | 1 EiB | 16 EiB |
| Online Grow | ✅ | ✅ | ✅ |
| Online Shrink | ❌ | ✅ | ✅ |
| Native Snapshots | ❌ | ❌ | ✅ |

```bash
# XFS
mkfs.xfs /dev/sdb1 && xfs_growfs /mnt && xfs_repair /dev/sdb1

# Ext4
mkfs.ext4 /dev/sdb1 && resize2fs /dev/sdb1 && e2fsck -f /dev/sdb1

# inode exhaustion
df -i
find / -xdev -printf '%h\n' | sort | uniq -c | sort -rn | head
```

### /etc/fstab Best Practices

```
# Always use UUID in production
UUID=xxxxxxxx  /app  xfs  defaults,noatime  0  0
# Add _netdev for network storage
UUID=xxxxxxxx  /data  ext4  defaults,_netdev  0  0
```

> ⚠️ A bad `/etc/fstab` entry can prevent boot. Always run `mount -a` after editing.

## 3.4 Multipath

```bash
dnf install device-mapper-multipath
mpathconf --enable --with_multipathd y
systemctl enable --now multipathd

# Key commands
multipath -ll                    # Show topology
multipath -f /dev/mapper/mpathX  # Remove ghost device
multipathd show paths format "%w %d %t %i %o %T %z"
```

## 3.5 NVMe

```bash
nvme list && nvme list-subsys
echo 120 > /sys/module/nvme_core/parameters/io_timeout

# NVMeoF
nvme discover -t tcp -a 192.168.1.100 -s 4420
nvme connect -t tcp -a 192.168.1.100 -s 4420 -n <nqn>
```

## 3.6 Software RAID (mdadm)

```bash
mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc
cat /proc/mdstat
mdadm --manage /dev/md0 --fail /dev/sdb --remove /dev/sdb --add /dev/sdd
```

## 3.7 iSCSI

```bash
dnf install iscsi-initiator-utils
iscsiadm -m discovery -t st -p 192.168.1.100
iscsiadm -m node -T <target_iqn> -p 192.168.1.100:3260 -l
iscsiadm -m node -T <target_iqn> -o update -n node.startup -v automatic
```

## 3.8 LUKS Encryption

```bash
cryptsetup luksFormat /dev/sdd
cryptsetup luksOpen /dev/sdd secure_data
mkfs.xfs /dev/mapper/secure_data

# Auto-mount
echo "secure_data /dev/sdd none" >> /etc/crypttab
echo "/dev/mapper/secure_data /secure xfs _netdev 0 0" >> /etc/fstab
```

---

# Chapter 4: Networking & Diagnostics

> 🎯 **Goal**: Configure network interfaces, manage firewalls, diagnose network issues, and tune network performance.

## 4.1 NetworkManager (nmcli)

```bash
nmcli device status
nmcli connection show

# Static IP
nmcli con mod eth0 ipv4.addresses 192.168.1.10/24
nmcli con mod eth0 ipv4.gateway 192.168.1.1
nmcli con mod eth0 ipv4.dns "8.8.8.8 1.1.1.1"
nmcli con mod eth0 ipv4.method manual
nmcli con up eth0
```

## 4.2 Bonding & VLAN

```bash
# Bond (active-backup)
nmcli con add type bond ifname bond0 mode active-backup bond.options "miimon=100"
nmcli con add type ethernet ifname eth0 master bond0
nmcli con add type ethernet ifname eth1 master bond0

# VLAN
nmcli con add type vlan ifname vlan100 dev eth0 id 100
nmcli con mod vlan100 ipv4.addresses 10.0.100.10/24 ipv4.method manual
```

| Mode | Name | Switch Config Required | Use Case |
|------|------|----------------------|----------|
| 0 | balance-rr | Yes | Load balancing |
| **1** | **active-backup** | **No** | **HA (most common)** |
| 4 | 802.3ad (LACP) | Yes | Bandwidth aggregation |
| 6 | balance-alb | No | Adaptive load balancing |

## 4.3 Firewall (firewalld)

```bash
firewall-cmd --list-all
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --permanent --zone=trusted --add-source=10.0.0.0/8
firewall-cmd --permanent --add-rich-rule=\
  'rule family="ipv4" source address="10.0.0.0/8" port port="22" protocol="tcp" accept'
firewall-cmd --reload
```

> 💡 RHEL 8+ uses **nftables** as the backend. firewalld is the recommended frontend.

## 4.4 DNS

```bash
# Client diagnostics
dig example.com A +short
dig @8.8.8.8 example.com MX
dig -x 1.2.3.4                # Reverse lookup

# BIND server
dnf install bind bind-utils
named-checkconf
named-checkzone example.com /var/named/example.com.zone
```

## 4.5 tcpdump

```bash
tcpdump -i eth0 -nn port 22
tcpdump -i eth0 -w /tmp/capture.pcap -c 5000
tcpdump -i eth0 host 10.0.0.1 and port 443
tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0'
tcpdump -i any -s 0 -A port 80                # Show HTTP plaintext
```

### TCP Flags

| Flag | Meaning | Context |
|------|---------|---------|
| SYN | Connection request | 3-way handshake step 1 |
| SYN-ACK | Connection accepted | 3-way handshake step 2 |
| ACK | Acknowledgment | Data transfer |
| FIN | Graceful close | 4-way teardown |
| RST | Forced reset | Error / port not listening |

## 4.6 Network Performance Tuning

```bash
cat > /etc/sysctl.d/99-network.conf << EOF
# Buffers
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# Connections
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 65535
net.ipv4.ip_local_port_range = 1024 65535

# TIME_WAIT
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15

# BBR congestion control (recommended)
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
EOF
sysctl -p /etc/sysctl.d/99-network.conf
```

### Jumbo Frames

```bash
nmcli con mod eth0 802-3-ethernet.mtu 9000
ping -M do -s 8972 <target>   # 8972 + 28 = 9000
```

### Diagnostic Tools

```bash
ss -tuln                    # Listening ports (replaces netstat)
ss -s                       # Connection stats
mtr <target>                # Continuous traceroute
ip -s link show eth0        # Interface stats (inc. errors)
ethtool -S eth0             # NIC detailed stats
```

---

# Chapter 5: Security & Compliance

> 🎯 **Goal**: Harden SSH, configure SELinux, set up auditing, and manage file permissions.

## 5.1 SSH Hardening

```bash
# /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
AllowUsers admin deploy

# Strong crypto only
KexAlgorithms curve25519-sha256,diffie-hellman-group16-sha512
Ciphers aes256-gcm@openssh.com,chacha20-poly1305@openssh.com
```

```bash
# Key management (Ed25519 recommended)
ssh-keygen -t ed25519 -C "admin@server"
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host

# Jump host
ssh -J jump-host target-host

# ~/.ssh/config
Host prod-*
    ProxyJump jump-host
    IdentityFile ~/.ssh/id_ed25519
```

## 5.2 SELinux

```bash
getenforce && sestatus

# Mode switching
setenforce 0    # Permissive (log only)
setenforce 1    # Enforcing
# Permanent: /etc/selinux/config → SELINUX=enforcing

# File context
semanage fcontext -a -t httpd_sys_content_t "/data/www(/.*)?"
restorecon -Rv /data/www

# Port context
semanage port -a -t http_port_t -p tcp 8443

# Boolean
setsebool -P httpd_can_network_connect on

# Troubleshooting
ausearch -m AVC -ts recent
sealert -a /var/log/audit/audit.log
audit2allow -a -M mypolicy && semodule -i mypolicy.pp
```

## 5.3 File Permissions

```bash
# Special permissions
chmod u+s /usr/bin/prog     # SUID
chmod g+s /shared           # SGID
chmod +t /tmp               # Sticky bit

# ACLs
setfacl -m u:deploy:rwx /data
setfacl -d -m u:deploy:rwx /data   # Default ACL (inheritance)
getfacl /data
setfacl -b /data                     # Remove all ACLs

# Immutable files
chattr +i /etc/resolv.conf
chattr +a /var/log/secure
lsattr /etc/resolv.conf
```

## 5.4 System Auditing (auditd)

```bash
dnf install audit && systemctl enable --now auditd

# Rules
auditctl -w /etc/passwd -p wa -k passwd_change
auditctl -w /etc/shadow -p wa -k shadow_access
auditctl -a always,exit -F arch=b64 -S execve -k exec_log

# Queries
ausearch -k passwd_change -ts today
aureport --summary
aureport --login --failed
```

## 5.5 User & Privilege Management

```bash
# Create user
useradd -m -s /bin/bash -G wheel deploy

# Password policy
chage -M 90 -m 7 -W 14 deploy

# Account locking
faillock --user username
faillock --user username --reset

# Sudo best practices (always use visudo)
deploy ALL=(ALL) /bin/systemctl restart httpd, /bin/systemctl restart nginx
Defaults logfile="/var/log/sudo.log"
Defaults log_input, log_output
```

## 5.6 Hardening Checklist

| Category | Action | Priority |
|----------|--------|----------|
| Auth | Disable root SSH login | 🔴 High |
| Auth | Key-only authentication | 🔴 High |
| Auth | Install fail2ban | 🔴 High |
| Privilege | Least-privilege sudo | 🔴 High |
| Firewall | Allow only required ports | 🔴 High |
| SELinux | Keep Enforcing mode | 🟡 Medium |
| Audit | Enable auditd for key files | 🟡 Medium |
| Encryption | LUKS for data partitions | 🟡 Medium |
| Patching | Regular security updates | 🟡 Medium |
| Integrity | Deploy AIDE | 🟢 Low |
| Scanning | OpenSCAP compliance scans | 🟢 Low |

```bash
# AIDE
dnf install aide && aide --init
mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
aide --check

# OpenSCAP
oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_cis \
  --results /tmp/scan.xml \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
```

---

# Chapter 6: Core Services

> 🎯 **Goal**: Configure, manage, and troubleshoot essential Linux services.

## 6.1 systemd Service Management

```bash
systemctl start|stop|restart|reload <service>
systemctl enable|disable <service>
systemctl status <service>
systemctl --failed
systemctl mask|unmask <service>
```

### Custom Service Unit

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
User=appuser
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/server
Restart=on-failure
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

## 6.2 Time Sync (Chrony)

```bash
# /etc/chrony.conf
server time.cloudflare.com iburst
server ntp.ubuntu.com iburst

chronyc sources -v
chronyc tracking
timedatectl set-timezone America/New_York
```

## 6.3 Logging (Rsyslog & journald)

```bash
# Remote syslog
# /etc/rsyslog.d/remote.conf
*.* @@log-server:514

# journald
journalctl -u sshd -f
journalctl --since "1 hour ago" -p err
journalctl --vacuum-size=500M
```

## 6.4 Web Servers

### Apache

```bash
dnf install httpd mod_ssl
systemctl enable --now httpd

# Virtual host with TLS
cat > /etc/httpd/conf.d/mysite.conf << 'EOF'
<VirtualHost *:443>
    ServerName www.example.com
    DocumentRoot /var/www/mysite
    SSLEngine on
    SSLCertificateFile /etc/pki/tls/certs/server.crt
    SSLCertificateKeyFile /etc/pki/tls/private/server.key
</VirtualHost>
EOF
httpd -t && systemctl reload httpd
```

### Nginx

```bash
dnf install nginx
# /etc/nginx/conf.d/mysite.conf
server {
    listen 443 ssl http2;
    server_name www.example.com;
    ssl_certificate     /etc/pki/tls/certs/server.crt;
    ssl_certificate_key /etc/pki/tls/private/server.key;
    root /var/www/mysite;
}
nginx -t && systemctl reload nginx
```

## 6.5 NFS

```bash
# Server
echo "/data/share 10.0.0.0/8(rw,sync,no_root_squash)" >> /etc/exports
exportfs -rav && systemctl enable --now nfs-server

# Client
mount -t nfs4 server:/data/share /mnt/nfs
# fstab: server:/data/share /mnt/nfs nfs4 defaults,_netdev 0 0

# Troubleshooting
showmount -e server && nfsstat -c
```

## 6.6 Samba

```bash
dnf install samba samba-client
smbpasswd -a username
systemctl enable --now smb nmb

# Client mount
mount -t cifs //server/shared /mnt/smb -o credentials=/root/.smbcred,_netdev
```

## 6.7 LDAP / AD Integration

```bash
dnf install sssd realmd adcli
realm join --user=admin ad.example.com
realm list
id admin@ad.example.com

# Troubleshooting
sssctl domain-status ad.example.com
sss_cache -E
```

## 6.8 KVM Virtualization

```bash
dnf install qemu-kvm libvirt virt-install
systemctl enable --now libvirtd

virt-install --name vm01 --ram 4096 --vcpus 2 \
  --disk path=/var/lib/libvirt/images/vm01.qcow2,size=50 \
  --os-variant rhel9.0 --cdrom /iso/rhel-9.4-dvd.iso \
  --network bridge=br0 --graphics vnc

virsh list --all
virsh start|shutdown|console vm01
virsh snapshot-create-as vm01 snap1
virsh setvcpus vm01 4 --live
```

## 6.9 Postfix

```bash
dnf install postfix && systemctl enable --now postfix

# /etc/postfix/main.cf
relayhost = [smtp.relay.com]:587
smtp_tls_security_level = encrypt

# Queue
postqueue -p && postqueue -f && postsuper -d ALL
```

---

# Chapter 7: Containers & Cloud Native

> 🎯 **Goal**: Master Podman container management, image building, networking, storage, and Kubernetes basics.

## 7.1 Why Podman?

RHEL 8+ uses **Podman** as the default container engine, replacing Docker.

| Feature | Podman | Docker |
|---------|--------|--------|
| Daemon | **None** (Daemonless) | Requires dockerd |
| Rootless | ✅ Native | Requires root |
| CLI Compatibility | 100% Docker-compatible | — |
| systemd Integration | Native | Requires extra config |
| Security | Higher (no privileged daemon) | Daemon runs as root |

```bash
dnf install podman podman-compose
podman --version
```

## 7.2 Container Operations

```bash
# Pull & Run
podman pull docker.io/library/nginx:latest
podman run -d --name web -p 8080:80 nginx
podman run -it --rm ubi9 /bin/bash

# Manage
podman ps -a
podman logs -f web
podman exec -it web /bin/bash
podman stop web && podman rm web

# Resource limits
podman run -d --name app --memory=512m --cpus=1.5 -p 3000:3000 myapp
```

## 7.3 Image Building

```dockerfile
# Containerfile
FROM registry.access.redhat.com/ubi9/ubi-minimal:latest
RUN microdnf install -y python3 && microdnf clean all
WORKDIR /app
COPY . .
EXPOSE 8000
USER 1001
CMD ["python3", "app.py"]
```

```bash
podman build -t myapp:1.0 .
podman push myapp:1.0 registry.example.com/myapp:1.0
```

### Best Practices

| Principle | Explanation |
|-----------|-------------|
| Minimal base image | `ubi-minimal` over `ubi`, Alpine over Ubuntu |
| Merge RUN layers | Fewer layers = smaller image |
| Non-root user | `USER 1001` |
| Multi-stage builds | Separate build and runtime environments |
| `.containerignore` | Exclude unnecessary files |

## 7.4 Networking & Storage

```bash
# Custom network
podman network create mynet
podman run -d --name db --network mynet postgres:16
podman run -d --name app --network mynet -p 8080:8080 myapp

# Volumes
podman volume create dbdata
podman run -d -v dbdata:/var/lib/postgresql/data postgres:16
podman run -d -v /host/path:/container/path:Z nginx   # SELinux: :Z
```

## 7.5 Rootless Containers

```bash
podman run --user 1000:1000 -d nginx

# Port binding below 1024 (rootless)
sysctl net.ipv4.ip_unprivileged_port_start=80
```

## 7.6 systemd Integration

```bash
podman generate systemd --name web --new --files
cp container-web.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now container-web
```

## 7.7 Podman Compose

```yaml
version: '3'
services:
  web:
    image: nginx:latest
    ports: ["8080:80"]
    volumes: ["./html:/usr/share/nginx/html:Z"]
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
    volumes: ["dbdata:/var/lib/postgresql/data"]
volumes:
  dbdata:
```

```bash
podman-compose up -d
podman-compose logs -f
podman-compose down
```

## 7.8 Container Troubleshooting

```bash
podman inspect web | jq '.[0].NetworkSettings'
podman stats
podman top web
podman history myapp:1.0

# Common issues:
# Container exits immediately → Check CMD/ENTRYPOINT, view logs
# Network unreachable → Check firewalld, podman network inspect
# Permission denied → SELinux (:Z), file perms, user namespace
# Disk full → podman system prune -a
```

## 7.9 Kubernetes Troubleshooting

```bash
kubectl get nodes
kubectl get pods -A
kubectl get events --sort-by='.lastTimestamp'

# Pod debugging
kubectl describe pod <pod>
kubectl logs <pod> [-c container] [-f]
kubectl exec -it <pod> -- /bin/bash

# Common Pod states:
# CrashLoopBackOff → App crash, check logs
# ImagePullBackOff → Registry auth issue
# Pending          → Insufficient resources or scheduling constraints
# OOMKilled        → Out of memory, increase limits

# Network
kubectl get svc && kubectl get ingress
kubectl run debug --image=busybox --rm -it -- /bin/sh

# Resources
kubectl top nodes && kubectl top pods
```

---

# Chapter 8: Shell & Automation

> 🎯 **Goal**: Write robust Bash scripts, master text processing tools, and automate with Ansible.

## 8.1 Bash Essentials

### Variables & Parameters

| Syntax | Function | Example |
|--------|----------|---------|
| `${var:-default}` | Default if empty | `${DB_HOST:-localhost}` |
| `${#var}` | String length | `${#filename}` |
| `${var//old/new}` | Global replace | `${path//\//-}` |
| `${var##*/}` | Extract filename | `/a/b/c.txt` → `c.txt` |
| `${var%/*}` | Extract directory | `/a/b/c.txt` → `/a/b` |

### Special Variables

| Var | Meaning |
|-----|---------|
| `$0` | Script name |
| `$@` | All arguments (preserves quoting) |
| `$#` | Argument count |
| `$?` | Last exit code |
| `$$` | Current PID |

### Tests & Conditions

```bash
[[ -f /etc/passwd ]]       # File exists
[[ -d /tmp ]]              # Directory exists
[[ -n "$var" ]]            # Non-empty
[[ "$str" =~ ^[0-9]+$ ]]  # Regex match
(( count > 10 ))           # Arithmetic
```

## 8.2 Production Script Template

```bash
#!/bin/bash
set -euo pipefail

readonly SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
readonly LOG="/var/log/$(basename "$0" .sh).log"
readonly LOCK="/tmp/$(basename "$0" .sh).lock"

log()  { echo "[$(date '+%F %T')] $*" | tee -a "$LOG"; }
die()  { log "ERROR: $*" >&2; exit 1; }

exec 200>"$LOCK"
flock -n 200 || die "Already running"

[[ $# -ge 1 ]] || die "Usage: $0 <arg>"

cleanup() { rm -f /tmp/$$_*; }
trap cleanup EXIT
trap 'die "Error at line $LINENO"' ERR

log "Starting..."
# ... business logic ...
log "Done."
```

## 8.3 Text Processing

### grep

```bash
grep -rn "error" /var/log/
grep -E "error|fatal" app.log
grep -v "^#" config.conf
grep -c "ORA-" alert.log
grep -A 3 -B 1 "FATAL" app.log
grep -P '\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}' access.log
```

### find

```bash
find /var/log -name "*.log" -mtime +30 -delete
find / -size +100M -type f -exec ls -lh {} \;
find / -perm -4000 -type f -ls           # SUID audit
find / -nouser -o -nogroup               # Orphaned files
```

### awk

```bash
awk -F: '{print $1, $7}' /etc/passwd
awk '$5 > 90 {print $1, $5"%"}' <(df -h)
awk '{count[$1]++} END {for(k in count) print k, count[k]}' access.log
```

### sed

```bash
sed 's/old/new/g' file.txt
sed -i '/^#/d; /^$/d' config.conf    # Remove comments and blanks
sed -n '10,20p' file.txt
```

## 8.4 Useful Pipelines

```bash
# Top 10 largest directories
du -sh /* 2>/dev/null | sort -rh | head -10

# IP hit count from access log
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head

# Real-time log monitoring
tail -f /var/log/messages | grep --color -iE "error|warn|fail"

# Batch rename
for f in *.JPEG; do mv "$f" "${f%.JPEG}.jpg"; done

# Parallel SSH
cat hosts.txt | xargs -P 10 -I {} ssh {} "hostname; uptime"
```

## 8.5 Ansible Quick Start

```bash
dnf install ansible-core

# Ad-hoc
ansible all -m ping -i inventory.ini
ansible web -m shell -a "uptime"
```

### Playbook Example

```yaml
---
- name: Deploy Web Server
  hosts: web
  become: yes
  tasks:
    - name: Install Nginx
      dnf: name=nginx state=latest

    - name: Start Nginx
      systemd: name=nginx state=started enabled=yes

    - name: Open firewall
      firewalld: service=http permanent=yes state=enabled immediate=yes
```

---

# Chapter 9: Performance Analysis & Tuning

> 🎯 **Goal**: Apply systematic performance analysis methodology across CPU, memory, disk IO, and network.

## 9.1 USE Method (by Brendan Gregg)

For each resource, check: **Utilization** + **Saturation** + **Errors**

| Resource | Utilization (U) | Saturation (S) | Errors (E) |
|----------|----------------|-----------------|------------|
| CPU | `mpstat`, `top` | Run queue (`vmstat r`) | `dmesg`, `perf` |
| Memory | `free -h` | Swap usage, OOM | OOM in `dmesg` |
| Disk IO | `iostat %util` | `await`, queue depth | `smartctl`, IO errors |
| Network | `sar -n DEV` | TCP retransmits | `ip -s link` errors |

### First 60 Seconds Checklist

```
1. uptime               → Load trend
2. dmesg -T | tail      → Kernel errors
3. vmstat 1 5            → CPU/memory/IO overview
4. iostat -dxm 1 5       → Disk IO
5. free -h               → Memory
6. sar -n DEV 1 5        → Network
7. top -bn1 | head -20   → Process ranking
8. ss -tuln              → Listening ports
```

## 9.2 CPU Analysis

```bash
uptime                         # Load averages
mpstat -P ALL 2 5              # Per-core stats
top -H -p <pid>                # Thread view
pidstat -u 2 5                 # Per-process CPU
perf top                       # Kernel hotspots
perf record -g -p <pid>        # Sampling
cpupower frequency-set -g performance   # Disable power saving
```

## 9.3 Memory Analysis

```bash
free -h
cat /proc/meminfo | grep -E "MemTotal|MemAvailable|Cached|SwapTotal|SwapFree"
ps aux --sort=-%mem | head -10
vmstat 2 5                     # Watch si/so (swap in/out)
dmesg -T | grep -i "oom"      # OOM killer
numactl --hardware             # NUMA topology
```

## 9.4 Disk IO Analysis

```bash
iostat -dxm 2 5
```

| Metric | Meaning | Alert Threshold |
|--------|---------|-----------------|
| `%util` | Device busy % | >70% investigate |
| `await` | Avg IO latency (ms) | SSD >5ms, HDD >20ms |
| `avgqu-sz` | Avg queue depth | >2 investigate |
| `r/s + w/s` | IOPS | Above device spec = bottleneck |

```bash
iotop -oP                      # IO by process
cat /sys/block/sda/queue/scheduler   # IO scheduler
echo mq-deadline > /sys/block/sda/queue/scheduler
```

| Scheduler | Best For |
|-----------|----------|
| `mq-deadline` | Databases (RHEL 8/9 default) |
| `none` / `noop` | SSD / NVMe / VMs |
| `bfq` | Desktop / interactive |

## 9.5 TuneD

```bash
tuned-adm list
tuned-adm profile throughput-performance
tuned-adm profile latency-performance
tuned-adm profile virtual-guest
tuned-adm active
```

## 9.6 Resource Limits

```bash
# ulimits
ulimit -a
# /etc/security/limits.conf
oracle soft nofile 65535
oracle hard nofile 65535

# cgroups v2
systemd-cgtop
systemctl set-property myapp.service CPUQuota=200% MemoryMax=2G
```

---

# Chapter 10: Diagnostic Tools & Methodology

> 🎯 **Goal**: Master enterprise diagnostic tools and systematic troubleshooting approaches.

## 10.1 Sosreport

```bash
sosreport
sos report --only-plugins=networking,filesys,logs
# Output: /var/tmp/sosreport-<hostname>-<date>.tar.xz
```

## 10.2 PCP (Performance Co-Pilot)

```bash
dnf install pcp pcp-system-tools
systemctl enable --now pmcd pmlogger

pmstat -s 5
pmval -T 10sec disk.dev.read_bytes
pcp atop
```

## 10.3 eBPF Tools (bcc / bpftrace)

eBPF is a revolutionary Linux kernel technology for safe, dynamic observability.

```bash
dnf install bcc-tools bpftrace

# Common bcc tools (/usr/share/bcc/tools/)
execsnoop          # Trace new processes
opensnoop          # Trace file opens
biolatency         # Block IO latency histogram
tcplife            # TCP connection lifecycle
tcpretrans         # TCP retransmissions
cachestat          # Page cache hit ratio
```

```bash
# bpftrace one-liners
bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args->filename)); }'
```

## 10.4 strace

```bash
strace -p <pid>                    # Attach to process
strace -c -p <pid>                 # Summary statistics
strace -e trace=network -p <pid>   # Network calls only
strace -e trace=file -p <pid>      # File operations only
```

## 10.5 Troubleshooting Methodology

### Five-Step Process

```
1️⃣ Gather    → What are the symptoms? When did it start? Recent changes?
2️⃣ Hypothesize → Top 3 most likely causes based on symptoms
3️⃣ Test       → Verify hypotheses, starting with the simplest
4️⃣ Fix        → Apply fix, document all steps taken
5️⃣ Document   → Write post-mortem, update knowledge base
```

### Common Troubleshooting Paths

**System won't boot**
```
Check GRUB → Check kernel → Check initramfs → Check fstab → Check systemd
```

**Disk full**
```
df -h → du -sh /* | sort -rh → find / -size +100M → lsof | grep deleted
```

**Network unreachable**
```
ip a → ping gateway → ping DNS → dig → traceroute → ss/tcpdump → firewall-cmd
```

**Process at 100% CPU**
```
top → strace -c -p <pid> → perf top -p <pid> → check application logs
```

**OOM Killed**
```
dmesg -T | grep oom → /proc/<pid>/status → check cgroup limits → tune app or add memory
```

## 10.6 Log Analysis

```bash
journalctl -p err -b                         # Errors since boot
grep "Failed password" /var/log/secure       # Auth failures
dmesg -T --level=err,warn | tail -50         # Kernel warnings
journalctl -u <service> --since today        # Service logs
```

---

# Appendix A: Quick Reference Sheets

## A.1 systemctl

| Command | Purpose |
|---------|---------|
| `systemctl start/stop/restart svc` | Manage service |
| `systemctl enable/disable svc` | Boot persistence |
| `systemctl status svc` | View status |
| `systemctl --failed` | List failed units |
| `systemctl daemon-reload` | Reload unit files |
| `systemctl mask/unmask svc` | Prevent/allow start |

## A.2 Filesystem Operations

| Operation | XFS | Ext4 |
|-----------|-----|------|
| Create | `mkfs.xfs` | `mkfs.ext4` |
| Grow | `xfs_growfs /mount` | `resize2fs /dev/x` |
| Repair | `xfs_repair /dev/x` | `e2fsck -f /dev/x` |
| Info | `xfs_info /mount` | `tune2fs -l /dev/x` |

## A.3 Performance Tools

| Tool | Purpose | Key Usage |
|------|---------|-----------|
| `top` / `htop` | Process monitor | `top -H -p <pid>` |
| `vmstat` | System overview | `vmstat 2 10` |
| `iostat` | Disk IO | `iostat -dxm 2` |
| `mpstat` | Per-CPU stats | `mpstat -P ALL 2` |
| `sar` | Historical data | `sar -u/-r/-d 2 10` |
| `perf` | Kernel profiling | `perf top`, `perf record` |
| `strace` | Syscall tracing | `strace -c -p <pid>` |

## A.4 Container Commands

| Operation | Command |
|-----------|---------|
| Run container | `podman run -d --name x -p 8080:80 image` |
| View logs | `podman logs -f x` |
| Enter container | `podman exec -it x /bin/bash` |
| Build image | `podman build -t name:tag .` |
| Resource usage | `podman stats` |
| Cleanup | `podman system prune -a` |
| Generate systemd | `podman generate systemd --name x --new` |

## A.5 Network Diagnostics

| Tool | Purpose |
|------|---------|
| `ss -tuln` | Listening ports |
| `ip a` / `ip r` | Addresses / Routes |
| `mtr <host>` | Continuous traceroute |
| `dig <domain>` | DNS query |
| `tcpdump -i eth0` | Packet capture |
| `ethtool eth0` | NIC info |
| `nmap -sT <host>` | Port scan |

---

## 📝 Copyright

**Linux Ops Fieldbook v1.0** © 2026 Bo Wang. All rights reserved.

This book is based on the author's years of production environment experience. All commands and configurations reference publicly available Linux distribution documentation and community best practices.

📧 Contact: v0279158888@gmail.com
🔗 GitHub: [github.com/bowang168](https://github.com/bowang168)
🔗 LinkedIn: [linkedin.com/in/bowang168](https://www.linkedin.com/in/bowang168)

---

*Last updated: March 2026 | Compatible with RHEL/Rocky/Alma/Oracle Linux 8.x & 9.x*
