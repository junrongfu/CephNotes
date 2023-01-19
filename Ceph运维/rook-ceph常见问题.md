## 一、osd数未达到预期数量

1.1 排查osd对应容器的日志

通过kubectl logs检查rook-ceph-osd-xxxx的输出日志，确认是否因为配置问题导致osd无法正常running

1.2 排查rook-ceph-osd-prepare-xxxx的相关日志

rook operator在创建rook-ceph-osd-xx这个deployment之前先创建rook-ceph-osd-prepare的job任务用于检查对应节点上是否满足创建osd的条件，如果不满足会输出日志，通过kubectl logs rook-ceph-osd-prepare-xxxx -n hxceph查看，如果需要清理磁盘见文档[容器化部署磁盘清理](http://172.20.200.191:8003/pages/viewpage.action?pageId=863470092)

1.3 operator重新调谐创建osd

如果osd如法正常修复，通过删除rook-ceph-osd-x对应的deployment和rook-ceph-osd-prepare-x对应的job，触发operator重新调谐创建osd

## 二、rook-ceph-osd-prepare容器调度所在节点资源不足

当前基于host模式的osd部署方式，必须有两个pod亲和到磁盘所在的节点，rook-ceph-osd-xxx和rook-ceph-osd-prepare-xxxx，由于rook-ceph-osd-prepare-xxx是job任务，执行完成的状态是Completed，rook operator触发调谐都会把老的job删掉重新创建job，如果节点上资源不足导致osd-prepare调度不上去，operator会重新调谐，直到调度成功。目前此问题有两种解决方案：

1. 保障磁盘所在节点上的资源充足，完成rook-ceph-osd-prepare-xxxx的正常调度
2. 迁移osd到资源充足的节点，迁移流程见[rook-ceph osd迁移文档](http://172.20.200.191:8003/pages/viewpage.action?pageId=883066478)

## 三、cephfs驱逐客户端

1. 进入rook-ceph-tools容器，通过ceph fs dump确认驱逐哪个文件系统的客户端，获取对应的mds

2. ceph tell mds.xxxx client ls 获取对应客户端的id

3. 执行ceph tell mds.0 client evict id=4305或ceph tell mds.0 client evict client_metadata.=4305进行驱逐处理

4. ceph的默认配置被驱逐后会进入黑名单，需要手动把用户从黑名单中删除，触发客户端重连，黑名单查看及处理命令：

   $ ceph osd blacklist ls

​              listed 1 entries

​              127.0.0.1:0/3710147553 2018-03-19 11:32:24.716146 

​          $ ceph osd blacklist rm 127.0.0.1:0/3710147553 

​              un-blacklisting 127.0.0.1:0/3710147553

详细文档见：https://drunkard.github.io/cephfs/eviction/

## 四、cephfs目录卡住

在paas平台部署cephfs时提供了mds的resource配置，此配置如果太小mds容器会产生oom导致业务容器卡住的问题，在配置前可以参考已部署容器对资源的使用情况