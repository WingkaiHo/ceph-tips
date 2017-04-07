## 1. PG 对应osd定位

### 1.1 所有pg对应osd定位

```
$ ceph osd

PG_STAT OBJECTS MISSING_ON_PRIMARY DEGRADED MISPLACED UNFOUND BYTES LOG DISK_LOG STATE        STATE_STAMP VERSION REPORTED 
0.0     0       0                  0        0         0       0     0   0        active+clean 2017-04-07 02:09:00.819406

 
VERSION REPORTED UP  UP_PRIMARY ACTING ACTING_PRIMARY LAST_SCRUB SCRUB_STAMP                LAST_DEEP_SCRUB DEEP_SCRUB_STAMP
0'0     76:103   [0] 0          [0]    0              0'0        2017-04-06 16:40:03.431738 0'0             2017-04-01 05:15:31.475322 


UP:     对应的就是这个pg所有osd-id
PG_STAT:对应pg的id

$ ceph pg map 0.0
osdmap e76 pg 0.0 (0.0) -> up [0] acting [0]

```

### 1.2 列举对应类型pg

```
$ ceph pg ls incomplete

也可以选择其他类型
$ ceph pg ls {active|clean|down|replay|scrubbing|degraded|inconsistent|peering|repair|recovering|backfill_wait|incomplete|stale|remapped|deep_    
 scrub|backfill|backfill_toofull|recovery_wait|undersized|activating|peered}
```

### 1.3 解决osd无法启动问题

  这个错误之前有处理经验，时间偏移过大引起认证不通过，登陆上osd对应的机器，检查发现时间偏移了几个小时，引起错误，检查发现ntp配置文件使用的是默认配置文件，至少这台没配置ntp，调整好时间，重启osd解决无法启动问题
```
verify_reply couldn’t decrypt with error: error decoding block for decryption
```

### 1.4 解决PG长期停留在incomplete状态

  这个状态一般是环境出现过特别的异常状况，PG无法完成状态的更新，这个只要没有OSD被踢出去或者损坏掉，就好办，这个环境的多个PG的数据是一致的，只是状态不对，这个就把PG对应的OSD全部停掉，然后用`ceph-objectstore-tool`进行mark complete, 然后重启osd，一般来说这个都是能解决了，没出意外，这个环境用工具进行了修复.

  例如修复0.10, 对应的osd 是0, 3, 7

- 把ceph进入维护模式
```
$ ceph osd set noout 
```

- 查找对应osd的host机器
```
$ ceph osd find  0
```

- 登录3个osd对应的host机器, 停止对应osd服务, 例如
```
$ systemctl stop ceph-osd@0
$ ceph-objectstore-tool --op mark-complete --pgid 0.10 --data-path /var/lib/ceph/osd/ceph-0 --journal-path /var/lib/ceph/osd/ceph-0/journal 
```

- 重新启动3个osd服务
```
$ systemctl start ceph-osd@0
```

- 退出维护模式
```
$ ceph osd unset noout
```

### 1.5 解决PG长期处于activaing状态
   上面的各种问题都处理过来了，到了最后一个，有一个PG处于activating状态，对于ceph来说真是一个都不能少，这个影响的是这个PG所在的存储池当中的数据，影响的范围也是存储池级别的，所以还是希望能够修复好，在反复重启这个pg的所在的osd后，发现这个pg总是无法正常，并且这个机器所在的OSD还会down掉，开始以为是操作没完成，需要很多数据要处理，所以增加了`osd_op_thread_suicide_timeout`的超时值，发现增大到180s以后还是会挂掉，然后报一堆东西，这个时候想起来还没去检查下这个PG是不是数据之前掉了，检查后就发现了问题，主PG里面的目录居然是空的，而另外两个副本里面的数据都是完整的并且一样的，应该是数据出了问题，造成PG无法正常
   停止掉PG所在的三个OSD，使用`ceph-objectstore-tool`进行pg数据备份，然后用`ceph-objectstore-tool`在主PG上删除那个空的pg，这里要注意不要手动删除数据，用工具删除会去清理相关的元数据，而手动去删除可能会残留元数据而引起异常，然后用`ceph-objectstore-tool`进行数据的导入，然后重启节点，还是无法正常，然后开日志看，发现是对象权限问题，用工具导入的时候，pg内的对象是root权限的，而ceph 启动的权限无法读取，手动给这个pg目录进行给予ceph的权限，重启osd，整个集群正常了.
 
  备份3个osd对应的pg, 如下命令
```
$ systemctl stop ceph-osd@0
$ ceph-objectstore-tool --op export --pgid 0.10 --data-path /var/lib/ceph/osd/ceph-0 --journal-path /var/lib/ceph/osd/ceph-0/journal --file osd-0-pg-0.10
```

  删除那个空的pg
```
$ systemctl stop ceph-osd@0
$ ceph-objectstore-tool --op remove --pgid 0.10 --data-path /var/lib/ceph/osd/ceph-0 --journal-path /var/lib/ceph/osd/ceph-0/journal
```

### 1.6 由于时间不同步问题导致osd无法启动

```
verify_reply couldn’t decrypt with error: error decoding block for decryption
```

这个错误之前有处理经验，时间偏移过大引起认证不通过，登陆上osd对应的机器，检查发现时间偏移了几个小时，引起错误，检查发现ntp配置文件使用的是默认配置文件，至少这台没配置ntp，调整好时间，重启osd解决无法启动问题.
   
