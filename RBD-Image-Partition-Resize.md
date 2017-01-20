## 1. Resize rbd image offline
   
   Use the xfs pratition for example:

### 1) Create rbd image size is 5G

```
 #rbd create volume1 --size 5G
```

### 2) Format the volume as xfs file system

```
   #rbd feature disable volume1  exclusive-lock object-map fast-diff deep-flatten
   #rbd map volume1
    /dev/rbd0
   #mkfs.xfs /dev/rbd0
```

### 3) Mount the device and get volume size

```
  #mkdir mnt
  #mount /dev/rbd0 mnt/
  #df -h
    ...
	/dev/rbd0                         5.0G   33M  5.0G   1% /root/mnt
    ...
```


### 4) Resize the size of rbd volume

```
  #umount mnt/
  #rbd unmap /dev/rbd0

  #rbd resize volume-test --size 10G
```

### 5) Mount the device after resize

```
   #rbd map volume1
    /dev/rbd0
   #mount /dev/rbd0 mnt/

   #df -h
   ...
   /dev/rbd0                         5.0G   33M  5.0G   1% /root/mnt
   ...
```

### 6) Update xfs file system 

```
#xfs_growfs mnt/

meta-data=/dev/rbd0              isize=256    agcount=9, agsize=162816 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=0        finobt=0
data     =                       bsize=4096   blocks=1310720, imaxpct=25
         =                       sunit=1024   swidth=1024 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=0
log      =internal               bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=8 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 1310720 to 2621440

#df -h
...
   /dev/rbd0                         5.0G   33M  5.0G   1% /root/mnt
...
```

   if the filesystem is ext4 use the command to update file system `resize2fs`
