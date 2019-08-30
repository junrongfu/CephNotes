### Ceph的核心组件

- [ ] Monitor：一个Ceph集群需要多个Monitor组成的小集群，它们通过Paxos同步数据，用来保存OSD的元数据；Ceph monitor(MON)组件通过一系列的map来跟踪整个集群的健康状态，包括OSD、MON、PG和CRUSH等组件的map；所有的集群节点都向monitor节点报告状态，并分享每一个状态变化的信息。一个monitor为每一个组件维护一个独立的map；monitor不存储实际数据；
- [ ] OSD：Object Storage Device，负责将实际的数据以对象的形式存储在每一个集群节点的物理磁盘驱动器中，一个Ceph集群一般有多个OSD；
- [ ] MDS：Ceph Metadata Server，是CephFS服务依赖的元数据服务；
- [ ] Object：Ceph最底层的存储单元是Object，对象包含一个标识符、二进制数据、和由名字/值对组成的元数据，元数据语义完全取决于 Ceph 客户端；
- [ ] PG：全称Placement Groups，是一个逻辑概念，一个PG包含多个OSD，引入这一层是为了更好地分配数据和定位数据；
- [ ] RADOS：全称Reliable Autonomic Distribute Object Store，是Ceph集群的精华，用户实现数据分配、Failover等集群操作；
- [ ] Librados：是Rados提供库，因为RADOS是协议很难直接访问，因此上层的RBD、RGW和CephFS都是通过Librados访问；
- [ ] CRUSH：是Ceph使用的数据分布算法，类似一致性哈希，让数据分配到预期的地方；
- [ ] RBD：全称RADOS Block Device，是Ceph对外提供的块设备服务；
- [ ] RGW：全称RADOS Gateway，是Ceph对外提供的对象存储服务，接口与S3和Swift兼容；
- [ ] CephFS：全称Ceph File System，提供了一个任意大小且兼容POSIX的分布式文件系统；

### 硬件需求

```shell
Processor：
Ceph 元数据服务器对 CPU 敏感，它会动态地重分布它们的负载，所以你的元数据服务器应该有足够的处理能力（如 4 核或更强悍的 CPU ）。 Ceph 的 OSD 运行着 RADOS 服务、用 CRUSH 计算数据存放位置、复制数据、维护它自己的集群运行图副本，因此 OSD 需要一定的处理能力（如双核 CPU ）。监视器只简单地维护着集群运行图的副本，因此对 CPU 不敏感；但必须考虑机器以后是否还会运行 Ceph 监视器以外的 CPU 密集型任务。例如，如果服务器以后要运行用于计算的虚拟机（如 OpenStack Nova ），你就要确保给 Ceph 进程保留了足够的处理能力，所以我们推荐在其他机器上运行 CPU 密集型任务。
Memory：
元数据服务器和监视器必须可以尽快地提供它们的数据，所以他们应该有足够的内存，至少每进程 1GB 。 OSD 的日常运行不需要那么多内存（如每进程 500MB ）差不多了；然而在恢复期间它们占用内存比较大（如每进程每 TB 数据需要约 1GB 内存）。通常内存越多越好。
网络：
使用万兆网络，且分离client和cluster的网络；
存储：
OSD Journal使用ssd（因为 Ceph 发送 ACK 前必须把所有数据写入日志（至少对 xfs 和 ext4 来说是），因此均衡日志和 OSD 性能相当重要）；
```

- [ ] CPU：mds >= 4 Cores；osd >= 2 Cores；mon >= 1cores；
- [ ] MEM：mds >= 1GB；osd >= 500MB(须将数据恢复时使用的资源计算在内！每进程每 TB 数据需要约 1GB 内存)
- [ ] 单个驱动器容量越大，其对应的osd所需内存就越大，特别是在重权衡、回填、恢复期间，根据经验，1TB的存储空间大约需要1GB内存；
- [ ] client与cluster网络隔离；
- [ ] 每个osd独占一个磁盘驱动器；
- [ ] 官方不推荐使用raid，会降低性能，若坚持使用，推荐raid0和jbod；
- [ ] 运行ceph的节点不要运行其他进程，且在一个节点上只运行一种类型的守护进程；
- [ ] 系统、数据和日志存储在不同的磁盘;

### 直接安装

```shell
配置软件源，所有节点执行；
# cat >/etc/yum.repos.d/ceph.repo<<EOF
[ceph]
name=ceph
baseurl=https://mirrors.aliyun.com/ceph/rpm-hammer/el7/x86_64/
gpgcheck=0
priority=1

[ceph-noarch]
name=cephnoarch
baseurl=https://mirrors.aliyun.com/ceph/rpm-hammer/el7/noarch/
gpgcheck=0
priority=1

[ceph-source]
name=Ceph source packages
baseurl=https://mirrors.aliyun.com/ceph/rpm-hammer/el7/SRPMS
enabled=0
gpgcheck=1
type=rpm-md
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc
priority=1
EOF

# yum makecache

# yum install ntp ntpdate ntp-doc

添加运行进程的用户，所有节点执行；
# useradd ceph
# echo 'ceph' | passwd --stdin ceph
# echo "ceph ALL = (root) NOPASSWD:ALL" > /etc/sudoers.d/ceph
# chmod 0440 /etc/sudoers.d/ceph
# 配置sshd可以使用password登录
# sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
# systemctl reload sshd
# 配置sudo不需要tty
# sed -i 's/Default requiretty/#Default requiretty/' /etc/sudoers

将所有节点添加至hosts文件；
# cat >>/etc/hosts<<EOF
10.205.0.138  yfb-0-138
10.205.0.139  yfb-0-139
10.205.0.140  yfb-0-140
# 此处hosts配置一定要与主机名保持一致！！！
EOF

安装程序包，所有节点执行；
# yum install -y ceph ceph-radosgw # 相当于执行了ceph-deploy install

添加防火墙规则，所有节点执行；
# iptables -A INPUT -i br0 -p tcp -s 10.205.0.0/16 --dport 6789 -j ACCEPT


部署集群，admin节点执行；
# yum install ceph-deploy -y
# su - ceph
# ssh-keygen -t rsa -P ''

# ssh-copy-id ceph@yfb-0-138
# ssh-copy-id ceph@yfb-0-139
# ssh-copy-id ceph@yfb-0-140

# mkdir ceph-cluster
# cd ceph-cluster
# ceph-deploy new ceph1
部署monitor，生成key；
# ceph-deploy --overwrite-conf mon create-initial
分发key到其他节点；
# ceph-deploy admin yfb-0-138 yfb-0-139 yfb-0-140

在节点创建目录；
# mkdir -pv /data/osd1
# chmod 777 -R /data/osd1
# chown ceph:ceph -R /data/osd1

添加osd；
# ceph-deploy osd prepare yfb-0-138:/data/osd1 yfb-0-139:/data/osd1 yfb-0-140:/data/osd1
# ceph-deploy osd activate yfb-0-138:/data/osd1 yfb-0-139:/data/osd1 yfb-0-140:/data/osd1

创建元数据服务；
# ceph-deploy mds create yfb-0-140
创建RGW；
# ceph-deploy rgw create yfb-0-139
RGW 例程默认会监听 7480 端口，可以更改该节点 ceph.conf 内与 RGW 相关的配置，如下：
[client]
rgw frontends = civetweb port=80
```

###  解决安装过程中遇到的问题

```shell
1、--> 解决依赖关系完成
错误：软件包：1:librbd1-0.94.10-0.el7.x86_64 (ceph)
          需要：liblttng-ust.so.0()(64bit)
错误：软件包：1:ceph-radosgw-0.94.10-0.el7.x86_64 (ceph)
          需要：libfcgi.so.0()(64bit)
错误：软件包：1:ceph-0.94.10-0.el7.x86_64 (ceph)
          需要：liblttng-ust.so.0()(64bit)
错误：软件包：1:ceph-common-0.94.10-0.el7.x86_64 (ceph)
          需要：libbabeltrace.so.1()(64bit)
错误：软件包：1:librados2-0.94.10-0.el7.x86_64 (ceph)
          需要：liblttng-ust.so.0()(64bit)
错误：软件包：1:ceph-common-0.94.10-0.el7.x86_64 (ceph)
          需要：libbabeltrace-ctf.so.1()(64bit)
错误：软件包：1:ceph-0.94.10-0.el7.x86_64 (ceph)
          需要：libleveldb.so.1()(64bit)
 您可以尝试添加 --skip-broken 选项来解决该问题
 您可以尝试执行：rpm -Va --nofiles --nodigest

 
# yum install -y yum-utils && yum-config-manager --add-repo https://dl.fedoraproject.org/pub/epel/7/x86_64/ && yum install --nogpgcheck -y epel-release && rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7 && rm /etc/yum.repos.d/dl.fedoraproject.org*
 

2、[ceph@yfb-0-140 ~]$ ceph health 
2019-01-17 15:01:59.790209 7f16e2c07700 -1 monclient(hunting): ERROR: missing keyring, cannot use cephx for authentication
2019-01-17 15:01:59.790219 7f16e2c07700  0 librados: client.admin initialization error (2) No such file or directory
Error connecting to cluster: ObjectNotFound
[ceph@yfb-0-140 ~]$ sudo chmod 755 /etc/ceph/ceph.client.admin.keyring

3、# ceph-deploy mon add yfb-0-140
[yfb-0-140][INFO  ] Running command: sudo ceph --cluster=ceph --admin-daemon /var/run/ceph/ceph-mon.yfb-0-140.asok mon_status
[yfb-0-140][ERROR ] admin_socket: exception getting command descriptions: [Errno 2] No such file or directory

# cd ceph-cluster
# vim ceph.conf
public_network = 10.205.0.140/16
# ceph-deploy --overwrite-conf config push yfb-0-138 yfb-0-139 yfb-0-140

4、[ceph@yfb-0-140 ~]$ ceph health 
HEALTH_WARN 1 pgs degraded; 1 pgs stuck degraded; 1 pgs stuck unclean; 1 pgs stuck undersized; 1 pgs undersized; recovery 14/129 objects degraded (10.853%)

osd进程无异常，磁盘空间不足，一定要确保磁盘空间充足；

5、osd init failed: (36) File name too long
Ceph官网建议使用XFS作为OSD存储数据的文件系统，但使用的文件系统是ext4，而ext4存储xattrs的大小有限制，使得OSD信息不能安全的保存；因此就有两种方法来解决这个问题： 
	a、修改Ceph配置文件的osd选项。将下面的信息添加到Ceph配置文件中global的section中，Ceph集群中，如果osd存储数据的文件系统是ext4的，都需要修改这个配置文件；然后重启对应的osd服务；

osd max object name len = 256 
osd max object namespace len = 64 

	b、 将文件系统改为XFS；

6、]# docker exec mon rbd map test --pool test
rbd: sysfs write failed
RBD image feature set mismatch. Try disabling features unsupported by the kernel with "rbd feature disable".
In some cases useful info is found in syslog - try "dmesg | tail".
rbd: map failed: (6) No such device or address
禁用当前系统内核不支持的feature：
]# docker exec mon rbd info test/test
rbd image 'test':
	size 1 GiB in 256 objects
	order 22 (4 MiB objects)
	id: 18166b8b4567
	block_name_prefix: rbd_data.18166b8b4567
	format: 2
	features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
	op_features: 
	flags: 
	create_timestamp: Fri Aug 23 13:16:06 2019
]# docker exec mon rbd feature disable test exclusive-lock, object-map, fast-diff, deep-flatten --pool test

rbd info显示的RBD镜像的format为2，Format 2的RBD镜像支持RBD分层，是实现Copy-On-Write的前提条件

```