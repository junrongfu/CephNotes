# 5. 单个Ceph节点宕机处理

----------

在某些情况下，如服务器硬件故障，造成单台 Ceph 节点宕机无法启动，可以按照本节所示流程将该节点上的 OSD 移除集群，从而达到 Ceph 集群的恢复。

### 5.1 单台 Ceph 节点宕机/磁盘恢复处理步骤

1. 登陆 ceph monitor 节点，查询 ceph 状态：

   `ceph health detail`

2. 将故障节点上的所有 osd 设置成 out，该步骤会触发数据 recovery, 需要等待数据迁移完成, 同时观察虚拟机是否正常：

   `ceph osd out osd_id`

3. 从 crushmap 将 osd 移除，该步骤会触发数据 reblance，等待数据迁移完成，同时观察虚拟机是否正常：

   `ceph osd crush remove osd_name`

4. 删除 osd 的认证： `ceph auth del osd_name`

5. 删除 osd ：`ceph osd rm osd_id`


#磁盘恢复流程
该流程主要是在已经有数据盘的服务器重装时，或者将数据盘迁移到其他服务器时的操作流程，在重装前和数据盘迁移前不需要前置操作

vgs # 列出当前的vg组
vgimport $vgid # 导入需要修复的vg组，vgid为上一步第一列数据
vgchange -ay $vgid # 激活vg组
lvs -o lv_tags | grep $vgid # 查看该vg对应的osd标签，需要查询到osd的编号和fsid，找到如下两个数据ceph.osd_id=$osdNum, ceph.osd_fsid=$fsid,
ceph-volume lvm activate $osdNum $fsid
ceph osd in $osdNum






