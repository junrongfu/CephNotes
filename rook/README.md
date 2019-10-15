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

- launch the toolbox.

```shell
# kubectl apply -f toolbox.yaml
```

- create object store and store user

```shell
# kubectl apply -f object.yaml
# kubectl apply -f object-user.yaml
```

- got the admin keyring, and dashboard user password, and s3 store user info

```shell
# kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o yaml | grep "password:" | awk '{print $2}' | base64 --decode
```

### cluster cleanup

```shell
# delete the data on hosts
#!/usr/bin/env bash
DISK="/dev/sdb"
# Zap the disk to a fresh, usable state (zap-all is important, b/c MBR has to be clean)
# You will have to run this step for all disks.
sgdisk --zap-all $DISK

# These steps only have to be run once on each node
# If rook sets up osds using ceph-volume, teardown leaves some devices mapped that lock the disks.
ls /dev/mapper/ceph-* | xargs -I% -- dmsetup remove %
# ceph-volume setup can leave ceph-<UUID> directories in /dev (unnecessary clutter)
rm -rf /dev/ceph-*
```

### ISSUE

1. launch osd needs more memory, at least 2048Mi，must reconfig the `cluster.yaml`.
2. recreate the rook cluster, first, deletes the directory of `/var/lib/rook`.
3. there are 3 nodes in my cluster, and 10 osds per nodes, but after create a rbd pool, the default pg and pgs are 8, it's too small, I adjust them to 512(2 steps, fitsrt to 256, next to 512).

```shell
# ceph osd pool get $POOL pg_num
# ceph osd pool get $POOL pgp_num

# ceph osd pool set $POOL pg_num 256
# ceph osd pool set $POOL pgp_num 256
```

4. in our cluster, OS CentOS 7.3, kernel version 4.4, and ceph version minic, there is a error occurs when i map the rbd image, it's "missing required protocol features missing 400000000000000". and run the below command to resolve it. but it's not recommended in product env, and must upgrade kernel.

```shell
ceph osd crush show-tunables
ceph osd crush tunables hammer
ceph osd crush reweight-all
```

