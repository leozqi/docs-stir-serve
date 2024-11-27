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
# /dev/sda
# /dev/sdb

# rio terminal not recognized by cfdisk?
export TERM=xterm
cfdisk /dev/sda
```

<img width="834" alt="Screenshot 2024-11-27 at 2 36 09 pm" src="https://github.com/user-attachments/assets/5aec5f41-8c6f-4cbd-9240-d4a1b3f2e5a0">
