osd迁移的处理流程先缩容，把osd从集群中删掉，拔盘然后插到指定的节点上再扩容。详细流程如下：

## 一、缩容osd

通过rook host-base模式部署的ceph集群下线osd前需要检查是否满足以下状态：

1. 确保下线后有足够的空间保存原来的数据
2. 确保剩余的osd和所有的pg状态都是active+clean
3. 如果要下线多个osd，需要等上一个osd下线完成后，pg状态都恢复为active+clean再下线其它的osd



### 1.1 手动把osd的副本数降为0

```
`kubectl -n rook-ceph scale deployment rook-ceph-osd-<ID> --replicas=0`
```

通过ceph osd df命令确认osd为down状态

### 1.2 停止ceph operator，防止对老的osd重新创建新的osd id

```
`kubectl -n rook-ceph scale deployment rook-ceph-operator --replicas=0`
```

### 1.3 开始清理osd

[osd-purge.yaml](#)

使用附件中的osd-purge.yaml自动执行osd的删除操作，有以下两点需要注意：

1. osd必须为down状态
2. job执行的时候不会等backfilling状态完成

此job主要完成以下功能：

1. 把osd从集群中删除
2. 删除osd对应的deployment

在执行创建job前需要修改yaml中的参数：

1. --osd-ids  : 此参数配置为需要迁移的osd id
2. --force-osd-removal： 默认为false，如果无法删除需要设置true强制删除，注：设置true之前确认下pg的状态是否满足强制删除的条件
3. job名字，如果job已经存在需要删除老的或者修改job名字

执行命令为：

```
`kubectl create -f osd-purge.yaml`
```



以上为自动删除osd，如果osd的状态比较复杂需要手工删除，参考文档[purge-the-osd-manually](https://rook.io/docs/rook/v1.7/ceph-osd-mgmt.html#purge-the-osd-manually)

## 二、扩容osd

扩容osd只需要更新cephCluster CR即可，rook实时监听CR的状态，如果有新的设备加入到集群中，会创建osd并加入到ceph集群中

### 2.1 更新cephCluster CR

更新cephCluster 中的value，把原来的osd配置删除，按照原来的格式添加新的硬盘。重新发布更新。

### 1.5 开启operator

```
`kubectl -n rook-ceph scale deployment rook-ceph-operator --replicas=2`
```

start operator开始对ceph cluster进行更新处理。

如果ceph集群的状态为ok，但是osd还未清理完成需要执行ceph osd crush rm osd.x手工清理。



官方参考文档[Ceph OSD Management](https://rook.io/docs/rook/v1.7/ceph-osd-mgmt.html)