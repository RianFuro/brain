- Increase "Physical" Disk Size (in VM manager)
- Increase logical disk size
- Increase LVM volume
- Resize FS

# Increase logical disk size

As far as I can tell, that's done by deleting the partition in question and then recreating it with more space. That was too scary for me though (#TODO: test that), so I used growpart instead.

On CentOS, this can be installed by:
```bash
yum install cloud-utils-growpart
```
and then used like so:
```bash
growpart /dev/sdX <partition-num>
```

# Increase LVM volume

Tell LVM the disk size changed:
```bash
pvresize /dev/sdX
```

Find the LVM volume path:
```bash
lvdisplay
# look for "LV Path"
```

Extend:
```bash
lvextend -l +100%FREE <lvm-path>
```

# Resize FS

This _should_ be just:
```bash
resize2fs <lvm-path>/<partition>
```
However if the system is live this may not work, since the filesystem is currently mounted.

This can be worked around with a different tool:
```bash
mount /mnt <lvm-path>/<partition>
xfs_growfs -d /mnt
umount /mnt
```