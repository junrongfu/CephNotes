## ceph 换盘扩容

### 调整时间

基础设施调整操作：工作日0点之后操作，或者非工作日

基础设施包括网络、主机系统、存储 / 备份系统、安全系统、以及机房动力环境等

### 调整规范

变更管理实现所有基础设施和应用系统的变更，变更管理应记录并对所有要求的变更进行分类，应评估变更请求的风险、影响和业务收益。其主要目标是以对服务最小的干扰实现有益的变更。

调整需要对可能影响的业务方进行通知。如果是关键系统的调整，需要全局通知。

### 调整方式

后续均通过工单系统进行操作，重要操作需要至少两个人一起确认后在执行

### 调整实例

换盘扩容可以抽象为几个元步骤的结合：1)删除OSD步骤 2）观察集群状态 3)IDC更换磁盘 4)新OSD上线步骤 5)观察集群状态

### 更换磁盘

1、 根据OSD ID 确认需要更换的磁盘对应的盘符(ceph-25 25为OSD对应的ID，sdf为服务器上对应的盘符；)

```
[root@ceph-81-204 ~]# lsblk |grep -B1 `ls -lrt /var/lib/ceph/osd/ceph-25 | grep block |awk -F "-" '{print $NF}'`
sdf                                                                             └─ceph--f014814a--8721--4347--9ebd--b307bab59e0d-osd--block--a87fccf6--88bf--481e--9757--8d5b74d3ca59 253:4    0  7.3T  0 lvm
```
2、 获取磁盘SN:
```
lsblk --nodeps -no serial /dev/sda
```
根据磁盘SN获取磁盘slot位置:
```
/opt/MegaRAID/MegaCli/MegaCli64 -PDlist -Aall | grep -B4 $SN
```

//2、根据盘符找到服务器上对应的磁盘id，scsi-0:0:4:0标识第4号磁盘id；
//```
//[root@ceph-81-204 ~]# ll /dev/disk/by-path/ |grep sdf
//lrwxrwxrwx 1 root root  9 7月   7 21:57 pci-0000:03:00.0-scsi-0:0:4:0 -> ../../sdf
//```

3、获取磁盘SN号提交磁盘更换工单[http://wos.myhexin.com/?#/process/create-ticket?processId=5](http://wos.myhexin.com/#/process/create-ticket?processId=5)

（从A服务器拔出，更换到B服务器工单：[http://wos.myhexin.com/?#/process/create-ticket?processId=244](http://wos.myhexin.com/#/process/create-ticket?processId=244)）

```
[root@ceph-81-204 ~]# smartctl --all /dev/sdf | grep "Serial Number"Serial Number:    WSD62F28
```

### 删除 OSD

要想缩减集群尺寸或替换硬件，可在运行时删除 OSD 。在 Ceph 里，一个 OSD 通常是一台主机上的一个 `ceph-osd` 守护进程、它运行在一个硬盘之上。
如果一台主机上有多个数据盘，你得逐个删除其对应 `ceph-osd` 。通常，操作前应该检查集群容量，看是否快达到上限了，确保删除 OSD 后不会使集群达到 `near full` 比率。

**警告：** 删除 OSD 时不要让集群达到 `full ratio` 值，删除 OSD 可能导致集群达到或超过 `full ratio` 值。

1、停止需要剔除的 OSD 进程，让其他的 OSD 知道这个 OSD 不提供服务了。停止 OSD 后，状态变为 `down` 。

```
ssh {osd-host}sudo systemctl stop ceph-osd@{osd-num}
```

2.将 OSD 标记为 `out` 状态，这个一步是告诉 mon，这个 OSD 已经不能服务了，需要在其他的 OSD 上进行数据的均衡和恢复了。

```
ceph osd out {osd-num}
```

执行完这一步后，会触发数据的恢复过程。此时应该等待数据恢复结束，集群恢复到 `HEALTH_OK` 状态，再进行下一步操作。

3.删除 CRUSH Map 中的对应 OSD 条目，它就不再接收数据了。你也可以反编译 CRUSH Map、删除 device 列表条目、删除对应的 host 桶条目或删除 host 桶（如果它在 CRUSH Map 里，而且你想删除主机），重编译 CRUSH Map 并应用它。

```
ceph osd crush remove {name}
```

该步骤会触发数据的重新分布。等待数据重新分布结束，整个集群会恢复到 `HEALTH_OK` 状态。

4.删除 OSD 认证密钥：

```
    ceph auth del osd.{osd-num}
```

5.删除 OSD 。

```
ceph osd rm {osd-num}#for exampleceph osd rm 1
```

6.卸载 OSD 的挂载点。

```
sudo umount /var/lib/ceph/osd/$cluster-{osd-num}
```

7.登录到保存 `ceph.conf` 主拷贝的主机。

```
    ssh {admin-host}    cd /etc/ceph    vim ceph.conf
```

8.从 `ceph.conf` 配置文件里删除对应条目。

```
    [osd.1]            host = {hostname}
```

9.从保存 `ceph.conf` 主拷贝的主机，把更新过的 `ceph.conf` 拷贝到集群其他主机的 `/etc/ceph` 目录下。

如果在 `ceph.conf` 中没有定义各 OSD 入口，就不必执行第 7 ~ 9 步。

### 新增OSD

1.初始化新的磁盘、OSD初始化，在deploy服务器上运行，vlan1的deploy服务器是192.168.215.89，# 指定目标机器及对应的块设备编号，如果更换的是旧盘，这一步操作有可能失败报错，此时最好用dd（或者分区工具）摧毁磁盘上残留的Raid或分区表信息，然后重新初始化尝试；

```
su - cephfsdsh /usr/local/bin/ceph-devops/deploy-osd.sh $hostname sdb,sdc,sde,...sdn 
# 验证方法ceph osd status | grep $hostname # 确认最后一列是 exists,up表示osd已经启动完毕
```

2.调整Crushmap，由于现在测试和vlan1的ceph都变为了rack为故障域，并且线上还没有开启自动加入osd，所以需要手动的将新的bucket加入进去，命令如下：

```
ceph osd crush add-bucket $hostname host # 该命令是新增一个host级的bucket，名称是$hostname，一般以目标服务器的hostname作为名称
ceph osd crush add osd.* $weight host=$hostname # 该命令是将osd加入到host中去，如果已经自动创建了这一步不需要，$weight为硬盘容量转换为T的数值
ceph osd crush move $hostname rack=$rackname # 该命令是把整个host移动到对应的rack下面
```
