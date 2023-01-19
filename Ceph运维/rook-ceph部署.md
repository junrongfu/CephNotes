## 一、helm部署

以下部署流程以在10.10.81.191上通过ansible-playbook部署到指定pass平台上为例，yaml文件见附件，部署过程分以下三步：

| playbook yaml文件                 | 命令                                                         | 安装说明                                     |
| --------------------------------- | ------------------------------------------------------------ | -------------------------------------------- |
| install-copy-rook-ceph-image.yaml | ansible-playbook /data/rook-ceph/install-copy-rook-ceph-image.yaml | 拷贝rook ceph依赖的相关镜像到pass平台的hub上 |
| install-rook-operator.yaml        | ansible-playbook /data/rook-ceph/install-rook-operator.yaml  | 安装rook operator到pass集群                  |
| install-rook-ceph-cluster.yaml    | ansible-playbook /data/rook-ceph/install-rook-ceph-cluster.yaml | 安装ceph集群到pass集群                       |

## 二、验证

### 2.1 组件完整性检查

通过kubectl -n rook-ceph get pod -o wide 检查是否包含以下pod组件：

csi-cephfsplugin-xxx
csi-cephfsplugin-provisioner-xxxx
csi-rbdplugin-provisioner-xxx
csi-rbdplugin-xxx
rook-ceph-mds-xxx
rook-ceph-mgr-a-xxx
rook-ceph-mon-a-xxx
rook-ceph-operator-xxx
rook-ceph-osd-0-xxx
rook-ceph-rgw-xxx

rook-ceph-operator-xxx必须是running状态其它组件才能正常启动

### 2.2 ceph集群状态检查

使用官方提供的Toolbox查看ceph状态(见文档：[ceph-toolbox](https://rook.io/docs/rook/v1.7/ceph-toolbox.html))

kubectl exec -it rook-ceph-tools-b56b4db8-fh4vf -n rook-ceph bash，返回结果

[root@rook-ceph-tools-b56b4db8-fh4vf /]# ceph status
cluster:
id: a83f35da-161b-4c5b-b274-5bc4c72ae9a5
health: HEALTH_OK

### 2.3 对象存储组件检查

kubectl -n rook-ceph get svc -l app=rook-ceph-rgw

返回cluster ip：

NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
rook-ceph-rgw-my-store ClusterIP 10.106.193.9 <none> 80/TCP 3d

执行：curl 10.106.193.9:80

返回结果为以下内容，说明对象存储安装成功：

<?xml version="1.0" encoding="UTF-8"?><ListAllMyBucketsResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/"><Owner><ID>anonymous</ID><DisplayName></DisplayName></Owner><Buckets></Buckets></ListAllMyBucketsResult>

## 三、新增磁盘到ceph集群中

当前默认配置使用的是自动发现机制，rook-ceph-operator-xxx会检查是否有可用的未格式化的磁盘或分区，如果有会自动加入到ceph集群

## 四、部署问题排查

### 4.1 官方常见问题

官方文档列出了[Ceph Common Issues](https://rook.io/docs/rook/v1.7/ceph-common-issues.html#ceph-common-issues)，优先借助排查

### 4.2 operator重启

日志报连接api server异常，如
failed to get pod: Get "[https://10.96.0.1:443/api/v1/namespaces/rook-ceph/pods/rook-ceph-operator-6cc8c88867-nrvfw](https://10.96.0.1/api/v1/namespaces/rook-ceph/pods/rook-ceph-operator-6cc8c88867-nrvfw)": dial tcp 10.96.0.1:443: i/o timeout
排查方式：
此问题和rook无关，优先找容器组协助排查，当前还没结果，pod换个node后运行正常

### 4.3 csi-cephfsplugin-xxx或csi-rbdplugin-xxx部署失败

csi-cephfsplugin-xxx或csi-rbdplugin-xxx是通过daemonSet部署到每个node上的，rook-ceph的helm包valule.yaml默认kubelet的目录是/var/lib/kubelet，当前我们系统是安装在/data/kubelet，因此会出现mount失败的情况

## 五、回退

### 5.1 回退ceph cluster

ansible-playbook /data/rook-ceph/uninstall-rook-ceph-cluster.yaml

卸载ceph cluster，执行过程大概5分钟，完成后输出结果会有ok返回，failed都为0，如果没有安装ceph cluster会报错，提示找不到删除的资源

### 5.2 回退operator

ansible-playbook /data/rook-ceph/uninstall-rook-operator.yaml

卸载operator，执行过程大概10分钟，完成后输出结果会有ok返回，failed都为0，如果没有安装operator会报错，提示找不到删除的资源



[install-rook-ceph-yaml.tar.gz](#)

官方部署文档：[prometheus-monitoring](https://rook.io/docs/rook/v1.7/ceph-monitoring.html#prometheus-monitoring)

## 一、prometheus部署流程

rook-ceph prometheus的部署主要包含以下三步：

1. rook-ceph cluster开启监控
2. Prometheus Operator部署
3. rook监控服务部署

## 二、部署细节

### 2.1 rook-ceph cluster开启监控

如果是helm安装需要修改cluster/charts/rook-ceph-cluster/valu.yaml文件中的monitoring开关：

monitoring:
\# requires Prometheus to be pre-installed
\# enabling will also create RBAC rules to allow Operator to create ServiceMonitors
    enabled: true

如果手动安装修改cluster/examples/kubernetes/ceph/cluster.yaml：

```
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
[...]
spec:
[...]
  monitoring:
    enabled: true
    rulesNamespace: "rook-ceph"
[...]
```

### 2.2 Prometheus Operator部署

提前下载好https://raw.githubusercontent.com/coreos/prometheus-operator/v0.40.0/bundle.yaml 修改镜像地址，镜像要提前下载：

spec:
    containers:
    \- args:
         \- --kubelet-service=kube-system/kubelet
         \- --logtostderr=true
         \- --config-reloader-image=[hub-paas.hexin.cn/ths/jimmidyson/configmap-reload:v0.3.0](http://hub-paas.hexin.cn/ths/jimmidyson/configmap-reload:v0.3.0)
         \- --prometheus-config-reloader=[hub-paas.hexin.cn/ths/cortexlabs/prometheus-config-reloader:v0.40.0](http://hub-paas.hexin.cn/ths/cortexlabs/prometheus-config-reloader:v0.40.0)
         \- --prometheus-default-base-image=[hub-paas.hexin.cn/ths/prom/prometheus](http://hub-paas.hexin.cn/ths/prom/prometheus)
     image: [hub-paas.hexin.cn/ths/bitnami/prometheus-operator:v0.40.0](http://hub-paas.hexin.cn/ths/bitnami/prometheus-operator:v0.40.0)

kubectl apply -f bundle.yaml

查看operator是否为running

### 2.3 rook监控服务部署

执行命令：

cd cluster/examples/kubernetes/ceph/monitoring
kubectl create -f service-monitor.yaml
kubectl create -f prometheus.yaml
kubectl create -f prometheus-service.yaml
kubectl create -f rbac.yaml

执行完成后执行kubectl -n rook-ceph get pod prometheus-rook-prometheus-0验证pod是否running

### 2.4 验证

上述流程执行完成后通过执行获取prometheus的参数进行验证，执行如下命令获取服务地址：

echo "http://$(kubectl -n rook-ceph -o jsonpath={.status.hostIP} get pod prometheus-rook-prometheus-0):30900"

返回：

http://10.220.0.101:30900

通过curl “http://10.220.0.101:30900/metrics”查看返回数据结果



## 一、部署说明

ceph部署支持的存储设备有以下三种类型：

- 未格式化的磁盘
- 未格式化的分区
- 未格式化lvm

当前使用的rook版本为1.7.9，不支持[host-based-cluster](https://rook.io/docs/rook/v1.8/ceph-cluster-crd.html#host-based-cluster)的模式使用lv，如果使用lv必须使用[pvc-based-cluster](https://rook.io/docs/rook/v1.8/ceph-cluster-crd.html#pvc-based-cluster)的方式进行部署，以下对这两种部署方案进行介绍：

## 二、部署流程

### 2.1 host-based-cluster部署

此方式的部署流程可以在yaml文件中直接使用未格式化的磁盘或分区，lvm暂不支持[#7967](https://github.com/rook/rook/pull/7967)，cluster.yaml文件如：

```
`cephClusterSpec：   storage：     nodes:       - name: "k8s-test-0-101"         devices:          - name: "sda" # 磁盘         resources: {}       - devices:          - name: "sdb1" # 分区         name: "docker-0-27"         resources: {}  `
```

### 2.2 pvc-based-cluster部署

cluster在部署中通过基于pvc的方式直接使用底层的lv作为osd的存储资源，cluster.yaml文件如下：

```
` storage:    storageClassDeviceSets:     - name: set1       count: 3       portable: false       encrypted: false       volumeClaimTemplates:       - metadata:           name: data         spec:           resources:             requests:               storage: 10Gi           # IMPORTANT: Change the storage class depending on your environment (e.g. local-storage, gp2)           storageClassName: lvm-simple-shared           volumeMode: Block           accessModes:             - ReadWriteOnce`
```

storageClassDeviceSets字段含义见官方文档[storage-class-device-sets](https://rook.io/docs/rook/v1.8/ceph-cluster-crd.html#storage-class-device-sets)

由于pvc使用的volumeMode是Block，而当前的storageClass lvm-simple-shared只能使用FileSystem的模式，需要新开发支持Block模式的sc。为了测试，手动进行创建测试流程如下：

#### 2.2.1 在空余磁盘上创建lv

```
`1、创建pv pvcreate /dev/sdc  2、创建卷组 vgcreate vmdisk /dev/sdc  3、创建逻辑卷 lvcreate -n lv_disk1 -L 10G vmdisk`
```

#### 2.2.2 手动创建pv和sc

```
`kind: StorageClass apiVersion: storage.k8s.io/v1 metadata:   name: rook-manual provisioner: kubernetes.io/no-provisioner volumeBindingMode: WaitForFirstConsumer --- apiVersion: v1 kind: PersistentVolume metadata:   name: rook-pv01 spec:   storageClassName: rook-manual   capacity:     storage: 10Gi   accessModes:     - ReadWriteOnce   persistentVolumeReclaimPolicy: Retain   volumeMode: Block   local:     path: /dev/disk/by-id/dm-name-vmdisk-lv_disk1   nodeAffinity:       required:         nodeSelectorTerms:           - matchExpressions:               - key: kubernetes.io/hostname                 operator: In                 values:                 - k8s-test-0-101  # 由于ceph的故障域为host，至少需要在三个不同host上创建三个pv绑定对应的lv  `
```

pv创建完成后，在cluster.yaml中直接使用rook-manual这个storageClassName即可，以下截图创建了7个pvc。

![AL002基础服务 > rook-ceph磁盘(osd)部署方案 > image2022-3-24_14-32-40.png](http://172.20.200.191:8003/download/attachments/824281041/image2022-3-24_14-32-40.png?version=1&modificationDate=1648103561000&api=v2)

## 一、ceph账号创建

新部署的cephfs提供存储服务前需要创建mount目录和ceph账号密码，操作如下：

### 1.1 进入tools容器，此容器中包含ceph命令，可以创建账号

kubectl exec -it rook-ceph-tools-xxxxx bash -n hxceph

### 1.2 创建指定的mount目录

先mount cephfs根目录，然后在根目录里面创建/mnt/mindgo，执行命令如下：

 mon_endpoints=$(grep mon_host /etc/ceph/ceph.conf | awk '{print $3}')

 my_secret=$(grep key /etc/ceph/keyring | awk '{print $3}')

获取mon_endpoints和my_secret

到宿主机上临时挂载下ceph根目录(tools容器中挂载会失败)
 mount -t ceph -o name=admin,secret=$my_secret $mon_endpoints:/ /tmp/mount_dir

挂载成功后进入/tmp/mount_dir创建目录/mnt/mindgo

### 1.3 创建cephfs账号，如mindgo，此账号对 /mnt/mindgo有读写权限

ceph fs authorize ceph-filesystem client.mindgo  /mnt/mindgo rw

### 1.4 更新secret到配置createCephfsPVPVC.yaml中

![AL002基础服务 > 基于cephfs创建pvc和pv > image2022-2-11_16-17-50.png](http://172.20.200.191:8003/download/attachments/791906389/image2022-2-11_16-17-50.png?version=1&modificationDate=1644567471000&api=v2)

### 1.5 更新ceph mon地址到createCephfsPVPVC.yaml中

![AL002基础服务 > 基于cephfs创建pvc和pv > image2022-2-11_16-18-11.png](http://172.20.200.191:8003/download/thumbnails/791906389/image2022-2-11_16-18-11.png?version=1&modificationDate=1644567492000&api=v2)

## 二、创建pvc和pv

业务方使用cephfs时存在不同namespace下的业务共享，需要在对应的namespace下创建pvc绑定到已创建的pv上，附件中的createCephfsPVPVC.yaml配置是两个命名空间下的pvc与pv的创建与绑定。

[createCephfsPVPVC.yaml](#)

## 三、使用pvc

[pvc使用](http://172.20.200.191:8003/pages/viewpage.action?pageId=829067597#cephfs网盘申请及使用文档-二、网盘使用)

## 四、验证

到任意能够访问通monitor的服务器上执行mount，验证能否挂载成功。

 mount -t ceph -o name=mindgo,secret=mindgo_secret monitor地址:/mnt/mindgo /tmp/mount_dir