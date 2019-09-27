- each node must preinstall lvm2, if not, after node reboot, osds will be in CrashLoopBackOff .

```shell
# yum install lvm2
https://github.com/rook/rook/issues/2591
```

- Label nodes with 'role=storage-node' and tolerate taints with a key of 'storage-node'.

```shell
# kubectl label nodes $CEPH_NODE role=storage-node
# kubectl taint nodes $CEPH_NODE storage-node=:NoSchedule
```

- Create the common resources that are necessary to start the operator and the ceph cluster.

```shell
# kubectl create -f common.yaml
```

- Create the Operator deployment.

```shell
# kubectl create -f operator.yaml
```

- Create Ceph storage cluster.

You need to change some configuration sections in cluster.yaml.

```shell
# kubectl create -f cluster.yaml
```

- Create storage pools.

```shell
# kubectl create -f pool.yaml
```

- Create StorageClass and CephBlockPool.

```shell
# kubectl create -f csi/rbd/storageclass.yaml
```

- Mount rbd out of kubernetes cluster.

```shell
[ceph]
name=ceph
baseurl=https://mirrors.aliyun.com/ceph/rpm-mimic/el7/x86_64/
gpgcheck=0
priority=1

[ceph-noarch]
name=cephnoarch
baseurl=https://mirrors.aliyun.com/ceph/rpm-mimic/el7/noarch/
gpgcheck=0
priority=1

[ceph-source]
name=Ceph source packages
baseurl=https://mirrors.aliyun.com/ceph/rpm-mimic/el7/SRPMS
enabled=0
gpgcheck=1
type=rpm-md
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc
priority=1
```

```shell
# yum install -y ceph-common
# rbd create foo –size 4096 [-m {mon-IP}] [-k /path/to/ceph.client.admin.keyring]
# rbd map foo --pool rbd --name client.admin [-m {mon-IP}] [-k /path/to/ceph.client.admin.keyring]
```

### ISSUE

1. launch osd needs more memory, at least 2048Mi，must reconfig the `cluster.yaml`.
2. recreate the rook cluster, first, deletes the directory of `/var/lib/rook`.