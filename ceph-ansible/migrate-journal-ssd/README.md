### 1. 编辑`inventory`文件, 需要更换journal到ssd机器

    建议每次只做一台机器

```
$ vim inventory

[osds]
node-osd
```

### 2. 编辑对应的host变量文件

    变量文件是每个ssd分区,对应到指定的哪个osd上

```
$ cp host_vars/hostname.yml.sample host_vars/node-osd.yml
$ vim  host_vars/node-osd.yml

sds_journal_migration:
 - device_name: sdc
   partitions:
   - index: 1
     size: 5120M
     osd_id: 0
   - index: 2
     size: 5120M
     osd_id: 1
```

 - 其中`device_name`填写ssd磁盘名称, `index:` 是ssd分区号, 如果ssd不为空, 分区号必须是当前末尾分区号+1, `osd_id`在ceph上osd编号, 此osd必须在这台机器上. 否者失败.
 - 文件命名是为修改机器的hostname

### 3. 执行脚本

```
$ ansible-playbook -i inventory -k migrate-journal-to-ssd.yml 
```
