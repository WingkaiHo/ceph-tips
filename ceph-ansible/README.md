##1. 环境准备

###1.1 环境描述
       操作系统是centos7    
####1.1.1环境使用五台机器ip地址如下

```
host1: 192.168.20.1
host2: 192.168.20.2
host3: 192.168.20.3
host4: 192.168.20.4
host5: 192.168.20.5
```

####1.1.2 角色描述

monitor: host1~host3
osd: host1~host6
client: host1~host6

####1.1.3 网络

public network 192.168.20.0/24
cluster network 192.168.21.0/24

###1.2 环境配置

####1.2.1 网络要求
    集群所有服务器可以正常连接外网下载rpm包,ansible自动连接外网下载ceph对应的rpm包, 以及ceph系统依赖rpm包.

####1.2.2 配置集群的ip对照表
   ip 对照表必须填写public netwrok段ip地址
```
$vim /etc/hosts

192.168.20.1 host1
192.168.20.2 host2
192.168.20.3 host3
192.168.20.4 host4
192.168.20.5 host5
```

####1.2.3 配置ansible ceph 角色列表

```
$vim  /etc/ansbile/hosts

[mons]
host1
host2
host3

[osds]
host1
host2
host3
host4
host5
host6

[clients]
host1
host2
host3
host4
host5
host6
```
### 1.3 ceph-ansible 下载

```
  git clone https://github.com/ceph/ceph-ansible.git

  yum install ansbile
```

##2.1 ceph-ansible 参数配置说明

###2.1.1 host_checking_checking

    配置为`host_key_checking`为False, 解决首次登录的时候需要配置known_hosts.
```
$vim ./ceph-ansible/ansible.cfg 
...
host_key_checking = False
...
```

###2.1.2 验证各个机器的ssh环境是否正常

    为了提高验证速度建议验证前取保sshd以下配置
```
$vim /etc/ssh/sshd_config

UseDNS no
```
    测试所有机器是否ssh是否运作正常
```
$ansible all -m ping --ask-pass
SSH password: 
```
 
###2.2 配置ceph-ansible参数

####2.2.1 激活group_vars目录变量文件
 
   激活site.yaml 以及group_vars files
```
$ cp site.yml.sample site.yml
$ cp group_vars/all.yml.sample group_vars/all.yml
$ cp group_vars/mons.yml.sample group_vars/mons.yml
$ cp group_vars/osds.yml.sample group_vars/osds.yml
...
...

```

####2.2.2 配置全局变量文件

```
$vim group_vars/all.yml

// Configure package origin
// 'upstream':使用第三方源 'distro':使用系统源 'local': 本地源
// 建议使用distro, centos7 需要在 centos_package_dependencies: 变量添加centos-release-ceph-jewel
// 如果使用其他版本使用upstream
// ceph_stable: true # use ceph stable branch
// ceph_mirror: http://mirrors.163.com/ceph/
// ceph_stable_key: http://mirrors.163.com/ceph/
// ceph_stable_release: jewel # ceph stable release
// ceph_stable_repo: "{{ ceph_mirror }}/rpm-{{ ceph_stable_release }}"
ceph_origin: 'distro' 

// Monitor 配置
// 需要配置monitor 使用的网卡
monitor_interface: eth0

// OSD 配置
// 日志盘大小默认5120MB
journal_size: 5120
// 外部网络 network kvm等外部访问集群ip段
public_network: 192.168.20.0/24
// 集群网络 ceph osd, monitor 通讯使用
cluster_network: 192.168.21.0/24
```

####2.2.3 配置OSD变量文件

1) OSD磁盘
I. 自动发现模式
   自动发现模式自动发现osd机器上面所有空磁盘`没有分过区的磁盘`,都自动加入到osd盘

```
$vim group_vars/osds.yml

osd_auto_discovery: true
```
II. 用户指定osd磁盘
 
   建议集群每台机器osd数目以及磁盘名称`/dev/sdb /deb/sdc`都一致, 否则磁盘列表变量`devices:`需要在/etc/ansible/hosts每个osd机器分别传入.
所有`devices`必须是磁盘,不能是磁盘分区.
```
$vim group_vars/osds.yml

osd_auto_discovery: false

// 配置集群所有osd
devices:
 - /dev/vdb
 - /dev/sdc
 - /dev/sdd
 - /dev/sde
```

2) 日志模式

I. journal和osd使用同一磁盘
   
   在osd对应的磁盘上划分一个分区作journal日志盘使用. 分区大小在全局变量文件`group_vars/all.yml`的关键字`journal_size`配置

``` 
$vim group_vars/osds.yml
...
journal_collocation: true
...
```

II. 多个osd对应多个journal盘
   多个journal可以是一个磁盘的多个分区. 目前还不支持输入一个ssd, 自动按`journal_size`大小自动把ssd分区给多个osd作日志盘使用. 

```
$vim group_vars/osds.yml
...
raw_multi_journal: true
raw_journal_devices:
  - /dev/sdf1
  - /dev/sdf2
  - /dev/sdf3
  - /dev/sdf4
... 
```

SSD自动分区脚本使用可以参考: 

- [journal ssd 分区自动化](make-ssd-journal-partition/README.md)

III. 使用目录当osd使用

```
$vim group_vars/osds.yml

osd_directory: true
# 必须填写目录, devices 关键字不用配置
osd_directories:
 - /var/lib/ceph/osd/mydir1
 - /var/lib/ceph/osd/mydir2
```

## 3.1 自动化安装集群

### 3.1.1 Screen 安装和配置

   为了防止网络中断导致终端运行的ansible脚本中断导致集群安成功, 我们应该选用screen, 如果没有screen, 需要安装.

```
$ yum install -y screen
```

### 3.1.2 在screen执行ansible脚本

   上面配置只需要在集群其中一台机器下载ceph-ansible, 并且做上面相关配置, 其他机器无需要安装任何客户端. 然后执行进入ceph-ansible

```
$ screen -R ceph-install
$ ansible-playbook site.yml -k
```

   退出screen, 脚本将在后台执行, 方法如下:

```
  按组合键CRTL+A之后再按组合键CRTL+D
```

  重新进入screen, 可以查看后台运行的任务

```
  $ screen -R ceph-install
```

## 3.2 自定义参数

   自定义参数可以配置配置文件`group_vars/all.yml`, 可以修改变量`ceph_conf_overrides`, 支持`global`, `osd`, `mon`等覆盖.

```
$ vim group_vars/all.yml
ceph_conf_overrides:
  global:
    osd_pool_default_size: 2 
  osd:
    osd_max_write_size = 512
    osd_client_message_size_cap = 2147483648
    osd_deep_scrub_stride = 131072
    osd_op_threads = 8
    osd_disk_threads = 4
    osd_map_cache_size = 1024
    osd_map_cache_bl_size = 128
    filestore_max_sync_interval = 15
    filestore_min_sync_interval = 10
    filestore_queue_max_bytes = 10485760
    filestore_queue_committing_max_ops = 5000
    filestore_queue_committing_max_bytes = 10485760000
    filestore_op_threads = 32
    filestore_max_inline_xattr_size = 254
    filestore_max_inline_xattrs = 6
    osd_max_backfills = 2
    osd_recovery_max_active = 2
    osd_recovery_op_priority = 4
```

- [自动化安装ustack failtrue-domain](ustack-cluster/README.md)
- [迁移journal磁盘到ssd](migrate-journal-ssd/README.md)
