# 5. 单个Ceph节点宕机处理

----------

在某些情况下，如服务器硬件故障，造成单台 Ceph 节点宕机无法启动，可以按照本节所示流程将该节点上的 OSD 移除集群，从而达到 Ceph 集群的恢复。

### 5.1 单台 Ceph 节点宕机

停机维护前需要先关闭重平衡相关的操作，防止在停机维护过程中进行大量数据平衡，命令如下
 
`sh /usr/local/bin/ceph-devops/stop-rebalance.sh`  # 在任意ceph服务器上执行该命令即可，该命令是集群级别的
### 验证方法
`ceph -s | grep nobackfill |grep norecover | grep norebalance | grep flag(s) set` # 返回有结果表明ok
然后关闭需要停机维护服务器上的所有ceph组件

`systemctl stop ceph-*.target`  # 在需要停机的服务器上执行
### 验证方法
`ps aux | grep -v grep | grep -E '(ceph-mon|ceph-osd|ceph-mds|ceph-mgr|radosgw)'` # 没有对应的进程表示服务都已经停止了


### 5.2 磁盘恢复处理步骤

该流程主要是在已经有数据盘的服务器重装时，或者将数据盘迁移到其他服务器时的操作流程，在重装前和数据盘迁移前不需要前置操作

`vgs` # 列出当前的vg组

`vgimport $vgid ` # 导入需要修复的vg组，vgid为上一步第一列数据


`vgchange -ay $vgid ` # 激活vg组


`lvs -o lv_tags | grep $vgid `# 查看该vg对应的osd标签，需要查询到osd的编号和fsid，找到如下两个数据ceph.osd_id=$osdNum, ceph.osd_fsid=$fsid,

`ceph-volume lvm activate $osdNum $fsid `

`ceph osd in $osdNum`

`ls -lrt /var/lib/ceph/osd/ceph-* | grep ceph | grep osd | grep var |awk -F "-" '{print $2}' |cut -d : -f 1 >/data/osd-id.txt `

`cat /data/osd-id.txt |while read osdid;do ceph auth add osd.$osdid osd 'allow *' mon 'allow rwx' -i /var/lib/ceph/osd/ceph-$osdid/keyring;done `

`vgs | awk '{print $i}' > /data/vgid.txt`

`cat /data/vgid.txt |while read vgid;do lvs -o lv_tags | grep $vgid |awk -F "=|," '{print $18,$16}' ;done > /data/osdid-fsid.txt`

`cat /data/osdid-fsid.txt|while read line;do ceph-volume lvm activate $line;done`






