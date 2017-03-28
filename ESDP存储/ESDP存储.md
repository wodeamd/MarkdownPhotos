# 概述
管理存储不同于管理计算资源。管理员可以在 ESDP 企业安全数字化平台 (Enterprise Secure Digital Platform) 中使用 Kubernetes 的存储卷 (persistent volume or PV) 给集群配置存储。 
使用存储卷声明 (persistent volume claims or PVCs)，用户可以请求存储卷资源而不必知道背后的存储介质细节。
  
存储卷声明 (以下可以和PVC互用) 附属于特定的工程，用户可以通过它使用存储卷 (以下可以和PV互用)。 PV 资源不会被限定到某一个工程，它在整个集群中可见并可以被任何工程使用。当一个 PV 被绑定到某个工程，它不能再被其它工程使用。

PV 可以通过 PV API 对象定义，代表一个已存在的网络存储。

存储的高可用性需要存储的提供者保证。

PVC 可以通过 PVC API 对象定义，代表开发者对存储的请求。

# 存储卷和存储卷声明的生命周期

PV 是集群里的资源。PVC 是对这些资源的请求和检查。PV 和 PVC 的使用有以下几个阶段：

## 配置

管理员可以提前在集群中配置一系 PV，这些 PV 是对集群中真实存在的存储的描述。


## 绑定

用户可以创建 PVC 请求一个 PV，在 PVC 中指定大小和访问模式。当系统中存在满足 PVC 请求的 PV 时，这两者之间会创建绑定关系。

PVC 会保持未绑定状态一直到有可用的PV。例如，当 PVC 请求 100G 存储时，即使系统中有很多 50G 的 PV，它们也不能被绑定到一起。当管理员在添加一个 100G 的 PV 后，新创建的 PV 会和 PVC 绑定。

## 使用

容器集将 PVC 作为一个卷来使用。ESDP 会检查 PVC 并将 PV 中的存储挂载为容器的卷。当 PV 支持多种访问模式，用户需要在 PVC 中指定使用的访问模式。

一旦一个 PVC 被绑定并被容器集使用， PVC 绑定的 PV 就属于容器集。用户可以在配置容器集是指定使用哪个 PVC，稍后会有实例演示配置文件。

## 释放

当不再需要存储时，用户可以删除 PVC 对象，ESDP 会试图重新声明 PVC 绑定的 PV 以便可以被其它 PVC 使用。重新声明之前的 PV 上的数据会一直保留。

## 重新声明

有两种重新声明策略，保持或回收。

保持允许管理员手动重新声明存储资源。有些存储卷支持”回收“重新声明策略，会删除存储卷中的所有数据并允许存储卷被其它存储卷声明使用。

# 存储卷

每个 PV 包含一个 “spec” 和 “status”，代表存储卷的规格和状态。 

*PV 对象的定义*
```
apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: pv0003
  spec:
    capacity:
      storage: 5Gi
    accessModes:
      - ReadWriteOnce
    persistentVolumeReclaimPolicy: Recycle
    nfs:
      path: /tmp
      server: 172.17.0.2
```

## 存储卷类型
 ESDP 支持以下存储卷类型：

* NFS

* HostPath

* GlusterFS

* Ceph RBD

* OpenStack Cinder

* AWS Elastic Block Store (EBS)

* GCE Persistent Disk

* iSCSI

* Fibre Channel	  

## 大小

每个 PV 会包含一个存储大小的描述，通过 “capacity” 属性配置。

当前，大小是唯一支持的资源属性，将来会增加其它属性例如 IOPS， 吞吐量 (throughput) 等。

## 访问模式
一个存储卷可以任意一种存储支持的访问模式挂载到一个主机。某种存储可能支持多种不同的挂载模式，但是在 PV 中必须明确指定一种。PV 支持配置多个访问模式。
例如，NFS 可以支持多个读写客户端 (multiple read/write clients)，但某个 PV 中可以指定只读访问模式 (read-only)。

当 ESDP 为 PVC 寻找匹配的 PV 时，会使用访问模式和大小作为匹配标准。PVC 里是用户请求的访问模式。然而，用户可能被赋予比请求更多的权限，但不是更少。例如，当请求的访问模式是 “RWO” 单系统存在的唯一 PV 的访问模式是 “RWO+ROX+RWX”，这个 PV 被认为是匹配的因为它支持 “RWO”。
ESDP 会首先尝试完全匹配。存储卷必须包含或多于请求的的访问模式。
存储卷的容量必须等于或大于请求的容量。当有两种存储卷 (NFS and ISCSI)都满足请求时，ESDP会随机选取一个分配给存储卷请求。

ESDP 会对存储卷按访问模式分组并在组内按大小排序，当用户请求存储卷时，会选取匹配模式组内大小最匹配 (等于或大于并最接近请求大小) 的存储卷。


ESDP 支持的访问模式:

 访问模式	| CLI 缩写		| 描述
---|---|---
ReadWriteOnce	| RWO	| 存储卷可以以 read-write 模式挂载到一个节点。
ReadOnlyMany	| ROX	| 存储卷可以以 read-only 模式挂载到一个节点
ReadWriteMany	| RWX	| 存储卷可以以 read-write 模式挂载到多个节点

>存储卷的访问模式是对存储的访问模式的描述，ESDP 不会检查真实的存储的访问模式。当使用存储不支持的访问模式，存储的提供者负责检查并返回运行时错误。
>
>例如，GCE 存储支持 ReadWriteOnce 和 ReadOnlyMany 访问模式。当用户想使用存储的 ROX 访问模式时，必须制定存储卷声明的访问模式为 read-only。


下面的表格中列出常用存储支持的访问模式:

*表格 1. 存储支持的访问模式*

存储 | 	ReadWriteOnce | 	ReadOnlyMany | 	ReadWriteMany
---|---|---|---
AWS EBS | X | - | -
Azure Disk | X | - | -
Ceph RBD | X | X | - | 
Fiber Channel | X | X | - | 
GCE Persistent Disk | X | - | -
GlusterFS | X | X | X
HostPath | X | - | - | 
iSCSI | X | X | -
NFS | X | X | X
Openstack Cinder | X | - | -


## 回收策略
当前支持的回收策略：


回收策略 |	描述
---|---
Retain | 手工重新声明
Recycle | 自动删除数据 (例如, rm -rf /<volume>/*)

>当前，只有 NFS 和 HostPath 支持 'Recycle' 回收策略.

## 阶段 (Phase))
一个存储卷可能处于一下 阶段 (Phase)：

 阶段 (Phase)	 | 描述
---|---
Available | 未绑定到任何存储卷声明。
Bound | 绑定到某个存储卷声明。
Released | 存储卷声明被删除，单存储还未被重新声明。
Failed | 自动重新声明失败。


# 存储卷声明

每个存储卷声明包含一个 “spec” 和 “status”，代表存储卷声明的规格和状态。

*存储卷声明定义*
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: myclaim
  annotations:
    volume.beta.kubernetes.io/storage-class: gold
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
```

 访问模式
存储卷声明使用和存储卷相同的访问模式定义。

## 资源
存储卷声明, 需要制定请求的存储资源的大小。

## 在容器集中使用
容器集通过将存储卷声明作为卷 (volume) 来使用存储。容器集必须和他使用的存储卷声明在同一个工程。存储卷会被挂载到主机，然后挂载到容器集。

```
kind: Pod
apiVersion: v1
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: dockerfile/nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim	  
```		