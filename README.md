# CephNAS_P
Simple commands to set up a NAS with CephFS

## 0. Preparation
System: Should work with any arm64 or x86_64 computer, preferrably with at least 4 storage devices  
OS: Fedora Server, or at least supporting nm-manager, podman and SELinux

Privileges: Most (if not all) of the following commands require root privileges, ``sudo su`` before starting instead of prepending everything with sudo. Remember to ``exit`` after finishing

## 1. Cephadm bootstap
Set static IP (adjust to the correct interface name and subnet, skip if server IP is already static)  
This configuration leaves DHCP in place and just adds an extra IP to the existing interface
```
nmcli con mod "enp4s0" +ipv4.addresses "192.168.8.44/24"
nmcli con up enp4s0
```
Install dependencies
```
dnf install -y podman cephadm ceph-common
```
Bootstrap
```
cephadm bootstrap --cluster-network 192.168.8.0/24 --mon-ip 192.168.8.44 --initial-dashboard-user admin --initial-dashboard-password ceph --allow-fqdn-hostname --single-host-defaults
```
Adopt all free devices (skip if more complex setup (with WAL, DB, etc.) is required)
```
cephadm shell
ceph orch apply osd --all-available-devices
```
Windows (especially through a VPN) needs the FQDN as the API-URL, the default is just the hostname (ceph vs. ceph.lan)
```
ceph dashboard set-grafana-api-url https://$(hostname -f):3000
ceph dashboard set-prometheus-api-host http://$(hostname -f):9095
ceph config set mgr mgr/prometheus/rbd_stats_pools "*"
```

## 2. WebUI
- Navigate to https://HOSTNAME.lan:8443 (address is also displayed in termial output of step 1)
- Login with ``admin``//``ceph`` and set a new password
- Create Storage Pools
  - cephfs_meta replicated (SSDs preferred)
  - cephfs_data replicated or erasure coded (with ec_overwrites)
- Create MDS (at least two)

## 3. Samba
Create Filesystem
```
ceph fs new cephfs cephfs_meta cephfs_data --force
```
Setup Snapshots (only changes between snapshots consume space, little overhead in most cases)  
Snapshot every hour, keep 24 hourly, 7 daily and 6 weekly snapshots
```
ceph mgr module enable snap_schedule
ceph fs snap-schedule add /net 1h --fs=cephfs
ceph fs snap-schedule retention add /net 24h7d6w --fs=cephfs
exit
```
Install Samba
```
dnf install -y samba
```
Samba Firewall rules
```
firewall-cmd --permanent --add-service=samba
firewall-cmd --reload
```
Mount CephFS on host
```
mkdir /mnt/cephfs
mount -t ceph admin@.cephfs=/ /mnt/cephfs
```
Set share permissions
```
mkdir /mnt/cephfs/net /mnt/cephfs/prox
chown admin:admin /mnt/cephfs/net
```
SELinux context for SMB share (repeat with other paths for additional mounts)
```
semanage fcontext -at samba_share_t '/mnt/cephfs/net(/.*)?'
restorecon -Rv /
```
SMB configuration
```
smbpasswd -a admin
cat <<'EOL' > /etc/samba/smb.conf
[cephfs]
    comment = ceph network share
    path = /mnt/cephfs/net
    read only = no
    inherit owner = yes
    inherit permissions = yes
EOL
```
Mount CephFS on boot and start SMB after (fstab not viable, CephFS is not up that early during boot)
```
cat <<'EOL' | crontab -
SHELL=/bin/bash
BASH_ENV=/etc/profile

@reboot until mountpoint -q /mnt/cephfs/; do mount -t ceph admin@.cephfs=/ /mnt/cephfs; sleep 10; done; systemctl start smb
EOL
```
