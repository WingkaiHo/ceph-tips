### 获取默认mon系统盘告警条件
```
mon-node$ ceph --admin-daemon /var/run/ceph/ceph-mon.node-12.asok config show | grep avail
"mon_data_avail_crit": "5",
"mon_data_avail_warn": "20",
```

- 如上面的返回, 默认低于20%发出warn,  低于5%发出严重告警
- `/var/run/ceph/ceph-mon.node-12.asok` 其中不同mon节点环境有不同的名称

## 修改告警条件

```
mon-node$ ceph --admin-daemon /var/run/ceph/ceph-mon.node-12.asok config set mon_data_avail_warn <new-value>
```
