##1. SSD 日志磁盘
     
     由于成本问题, ceph集群使用一个ssd盘作为多个osd的journal盘使用, 所以一个ssd分成多个区, 提供给不同的osd使用. ceph-ansible集群安装, 没有ssd分区功能, 如果一个ssd, 提供给多个osd使用时候需要用这个脚本先把ssd分区


##2. 编辑分区布局

###2.1 分区布局默认

   如果所有`osds`组的机器分区布局是一致的, 可以在`host_vars/default.yml`定义分区布局, 例如:

```
$ cp host_vars/default.yml.sample
$ vim host_vars/default.yml

// 例如把sde, sdf两个ssd进行分区, 每个分区大小10G. 分区大小必须大于 ceph osd_journal_size

devices:
 - device_name: sde
   partitions:
   - index: 1
     size: 10G
     type: journal
   - index: 2
     size: 10G
     type: journal
 - device_name: sdf
   partitions:
   - index: 1
     size: 10G
     type: data
   - index: 2
     size: 10G
     type: journal
```

   `osds`组的机器默认是在文件`/etc/ansible/hosts`的`[osds]`标签下的所有机器

###2.2 有些机器分布布局不一样

   如果某些机器不需要分区, 在host_vars目录下, 必须添加一个对应hostname的yml文件, 把devices变量置空, 否者是按照默认的列表进行分区

```
$vim host_vars/hostname.yml

devices: []
```

   如果某些机器, 需要使用其他分区布局, 必须添加一个对应hostname的yml文件, 否者是按照默认的列表进行分区, 例如:

```
devices:
 - device_name: sdg
   partitions:
   - index: 1
     size: 5G
     type: journal
   - index: 2
     size: 5G
     type: journal
 - device_name: sdh
   partitions:
   - index: 1
     size: 5G
     type: data
   - index: 2
     size: 5G
     type: journal
```
