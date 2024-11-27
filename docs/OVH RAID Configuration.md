This follows:

- https://community.hetzner.com/tutorials/freebsd-openzfs-via-linux-rescue
- 

Let's setup OVH RAID with BSD using their Bring Your Own Image (BYOI).
Relevant docs:

- https://help.ovhcloud.com/csm/en-ca-dedicated-servers-bringyourownimage?id=kb_article_view&sysparm_article=KB0030285

It was hard to set up using a FreeBSD cloud image, so let's boot into rescue mode and use netinstall.


```sh
/sbin/modprobe zfs
modinfo zfs | grep version

# version: 2.1.11-1
# srcversion: 8081FD700719F8F0FB60578

# get base system and kernel for latest 14.1-RELEASE
curl -O http://ftp2.de.freebsd.org/pub/FreeBSD/releases/amd64/14.1-RELEASE/base.txz
curl -O http://ftp2.de.freebsd.org/pub/FreeBSD/releases/amd64/14.1-RELEASE/kernel.txz

lsblk
# Using a 2x480 G disk configuration
# /dev/sda
# /dev/sdb

# rio terminal not recognized by cfdisk?
export TERM=xterm
cfdisk /dev/sda
```

<img width="834" alt="Screenshot 2024-11-27 at 2 36 09 pm" src="https://github.com/user-attachments/assets/5aec5f41-8c6f-4cbd-9240-d4a1b3f2e5a0">

Do the same for the other disk

```sh
lsblk

# NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
# sda      8:0    0 447.1G  0 disk
# |-sda1   8:1    0   100M  0 part
# |-sda2   8:2    0    32G  0 part
# `-sda3   8:3    0   415G  0 part
# sdb      8:16   0 447.1G  0 disk
# |-sdb1   8:17   0   100M  0 part
# |-sdb2   8:18   0    32G  0 part
# `-sdb3   8:19   0   415G  0 part
```

Configure ZFS

```sh
zpool create -m none -O atime=off -O compression=on rpool mirror /dev/sda3 /dev/sdb3
zfs create rpool/ROOT
zfs create -o mountpoint=legacy rpool/ROOT/FreeBSD
zpool set bootfs=rpool/ROOT/FreeBSD rpool
```

Copy BSD files over to new pool

```sh
apt -y install libarchive-tools
# get bsdtar bin
mount -t zfs rpool/ROOT/FreeBSD /mnt
bsdtar -x -C /mnt -f base.txz
bsdtar -x -C /mnt -f kernel.txz
```

Prepare EFI

```sh
mkfs.fat -s 1 -F 32 /dev/sda1
mkfs.fat -s 1 -F 32 /dev/sdb1
mkdir -p /efi /efi2
mount -t vfat /dev/sda1 /efi
mount -t vfat /dev/sdb1 /efi2
mkdir -p /efi/EFI/BOOT /efi2/EFI/BOOT
cp /mnt/boot/loader.efi /efi/EFI/BOOT/BOOTX64.efi
cp /mnt/boot/loader.efi /efi2/EFI/BOOT/BOOTX64.efi
```

Add bootloader configuration using <https://man.freebsd.org/cgi/man.cgi?loader.conf(5)>

```conf
zfs_load="YES"
vfs.root.mountfrom="zfs:rpool/ROOT/FreeBSD"
autoboot_delay="3"
kern.geom.label.disk_ident.enable="0"
```

Add autodhcp:

```sh
vi /mnt/etc/rc.d/autodhcp
chmod +x /mnt/etc/rc.d/autodhcp
```

```conf
#!/bin/sh

# PROVIDE: autodhcp
# BEFORE: NETWORKING netif routing hostname
# REQUIRE: mountcritlocal mdinit
# KEYWORD: FreeBSD

. /etc/rc.subr

name=autodhcp
rcvar=autodhcp_enable

load_rc_config $name

: \${autodhcp_enable:="NO"}

start_cmd="autodhcp_start"
stop_cmd=":"

autodhcp_start()
{
        _dif=\$(/sbin/ifconfig -l | /usr/bin/sed -E 's/lo[0-9]+//g')
        for i in \$_dif; do
                echo "ifconfig_\$i=\"DHCP\"" >> /etc/rc.conf.d/network
        done
}

load_rc_config \$name
run_rc_command "\$1"
```

Configure rc.conf

```
cat << EOF > /mnt/etc/rc.conf
hostname="ovh-bsd1.in.stir.software"
zfs_enable="YES"
sshd_enable="YES"
autodhcp_enable="YES"
```

Configure SSH

```sh
mkdir -p /mnt/root/.ssh
chmod 0400 /mnt/root/.ssh

# Enable root login without password
echo "PermitRootLogin without-password" >> /mnt/etc/ssh/sshd_config

# Add public key
echo "SERVER PUBLIC KEY ssh-ed25519..." >> /mnt/root/.ssh/authorized_keys
```
