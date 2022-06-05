#### Migrating LVM and Volume group from one machine to another




#### Creating a new LVM on additional hard disk /dev/xvdb

```
[root@ip-172-31-16-207 ~]# fdisk /dev/xvdb

Welcome to fdisk (util-linux 2.30.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0xc754a861.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 
First sector (2048-16777215, default 2048): 
Last sector, +sectors or +size{K,M,G,T,P} (2048-16777215, default 16777215): +4G

Created a new partition 1 of type 'Linux' and of size 4 GiB.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

[root@ip-172-31-16-207 ~]# lsblk
NAME    MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
xvda    202:0    0   8G  0 disk 
└─xvda1 202:1    0   8G  0 part /
xvdb    202:16   0   8G  0 disk 
└─xvdb1 202:17   0   4G  0 part 
xvdc    202:32   0   8G  0 disk 

[root@ip-172-31-16-207 ~]# pvs

[root@ip-172-31-16-207 ~]# pvcreate /dev/xvdb1
  Physical volume "/dev/xvdb1" successfully created.
  
[root@ip-172-31-16-207 ~]# pvs
  PV         VG Fmt  Attr PSize PFree
  /dev/sdb1     lvm2 ---  4.00g 4.00g
[root@ip-172-31-16-207 ~]# vgcreate vg00 /dev/xvdb1

  Volume group "vg00" successfully created
[root@ip-172-31-16-207 ~]# vgs
  VG   #PV #LV #SN Attr   VSize  VFree 
  vg00   1   0   0 wz--n- <4.00g <4.00g
  
[root@ip-172-31-16-207 ~]# lvcreate -L +3G -n lv_backup vg00
  Logical volume "lv_backup" created.
  
[root@ip-172-31-16-207 ~]# lvs
  LV        VG   Attr       LSize Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv_backup vg00 -wi-a----- 3.00g   
  
[root@ip-172-31-16-207 ~]# lvs -a -o +devices
  LV        VG   Attr       LSize Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert Devices     
  lv_backup vg00 -wi-a----- 3.00g                                                     /dev/sdb1(0)
 ```
 
 #### Mounting the LVM to /backup directory
 
 ```
  
[root@ip-172-31-16-207 ~]# mkdir /backup

[root@ip-172-31-16-207 ~]# lsblk
NAME               MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
xvda               202:0    0   8G  0 disk 
└─xvda1            202:1    0   8G  0 part /
xvdb               202:16   0   8G  0 disk 
└─xvdb1            202:17   0   4G  0 part 
  └─vg00-lv_backup 253:0    0   3G  0 lvm  
xvdc               202:32   0   8G  0 disk 

[root@ip-172-31-16-207 ~]# mkfs.ext4 /dev/vg00/lv_backup 
mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
196608 inodes, 786432 blocks
39321 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=805306368
24 block groups
32768 blocks per group, 32768 fragments per group
8192 inodes per group
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done 

[root@ip-172-31-16-207 ~]# mount /dev/vg00/lv_backup /backup/

[root@ip-172-31-16-207 ~]# lsblk
NAME               MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
xvda               202:0    0   8G  0 disk 
└─xvda1            202:1    0   8G  0 part /
xvdb               202:16   0   8G  0 disk 
└─xvdb1            202:17   0   4G  0 part 
  └─vg00-lv_backup 253:0    0   3G  0 lvm  /backup
xvdc               202:32   0   8G  0 disk 
[root@ip-172-31-16-207 ~]# df -TH
Filesystem                 Type      Size  Used Avail Use% Mounted on
devtmpfs                   devtmpfs  497M     0  497M   0% /dev
tmpfs                      tmpfs     506M     0  506M   0% /dev/shm
tmpfs                      tmpfs     506M  488k  506M   1% /run
tmpfs                      tmpfs     506M     0  506M   0% /sys/fs/cgroup
/dev/xvda1                 xfs       8.6G  1.7G  7.0G  20% /
tmpfs                      tmpfs     102M     0  102M   0% /run/user/1000
tmpfs                      tmpfs     102M     0  102M   0% /run/user/0
/dev/mapper/vg00-lv_backup ext4      3.2G  9.5M  3.0G   1% /backup


[root@ip-172-31-16-207 ~]# cd /backup/
[root@ip-172-31-16-207 backup]# touch {1..10}.txt
[root@ip-172-31-16-207 backup]#
```
#### Before migrating the LVM and Volume group we have to first disable the Volume group

```
[root@ip-172-31-16-207 lvm]# vgs
  VG   #PV #LV #SN Attr   VSize  VFree   
  vg00   1   1   0 wz--n- <4.00g 1020.00m
  
[root@ip-172-31-16-207 lvm]# vgchange -a n vg00
  0 logical volume(s) in volume group "vg00" now active
```

#### Exporting the Volume group

```
  [root@ip-172-31-16-207 lvm]# vgexport vg00
  Volume group "vg00" successfully exported
```

#### Once the disk is attached to the new machine, importing the Volume group

```

  [root@ip-172-31-16-207 lvm]# vgimport -a 
  Volume group "vg00" successfully imported
```
#### Activating the Logical volume

```
  
  root@ip-172-31-16-207 lvm]# vgchange -a y 
  1 logical volume(s) in volume group "vg00" now active
  
  
  [root@ip-172-31-16-207 lvm]# lvs
  LV        VG   Attr       LSize Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv_backup vg00 -wi-a----- 3.00g                                                    
[root@ip-172-31-16-207 lvm]# lvs -a -o +devices
  LV        VG   Attr       LSize Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert Devices     
  lv_backup vg00 -wi-a----- 3.00g                                              /dev/sdb1(0)
  
  ```
  
#### Mounting the LVM to /backup directory in the new machine

```
  
  
[root@ip-172-31-16-207 lvm]# mount /dev/vg00/lv_backup /backup/
[root@ip-172-31-16-207 lvm]# df -TH
Filesystem                 Type      Size  Used Avail Use% Mounted on
devtmpfs                   devtmpfs  497M     0  497M   0% /dev
tmpfs                      tmpfs     506M     0  506M   0% /dev/shm
tmpfs                      tmpfs     506M  488k  506M   1% /run
tmpfs                      tmpfs     506M     0  506M   0% /sys/fs/cgroup
/dev/xvda1                 xfs       8.6G  1.7G  7.0G  20% /
tmpfs                      tmpfs     102M     0  102M   0% /run/user/1000
tmpfs                      tmpfs     102M     0  102M   0% /run/user/0
/dev/mapper/vg00-lv_backup ext4      3.2G  9.5M  3.0G   1% /backup
```


  
  
  
  

