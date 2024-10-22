**官方文档**：https://kubernetes.io/zh/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/

**注意**
- 本文仅适用于kubeadm安装的高可用集群升级
- kubeadm安装的非高可用集群建议先升级为高可用集群
- 在升级kubernetes集群版本前，一定要看完[【集群升级-理论指南】](https://mp.weixin.qq.com/s/JsQIsCIA2tqChmA4T098Bw)
- 每个版本升级过程可能存在差异，请参照[**官方文档**](https://kubernetes.io/zh/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

### 准备
- 阅读[发行型说明](https://git.k8s.io/kubernetes/CHANGELOG/CHANGELOG-1.20.md)，了解目标版本特性，核心组件兼容性，bugfix等
- 集群应使用静态的控制平面和etcd Pod或者外部etcd
- 备份etcd，以及其他重要组件。**kubeadm upgrade**不会影响你的工作负载，只会涉及 Kubernetes 内部的组件，但备份终究是好的。
- 交换分区必须禁用
- 确认集群第三方组件与目标版本的兼容性，在升级完成节点的核心组件后，可能需要手动升级。


升级后，因为容器规约的哈希值已更改，所有容器都会被重新启动。

**节点升级顺序**
-  升级主控制平面节点
-  升级其他控制平面节点
-  升级工作节点

### 集群信息

名称 | 地址 | 作用 | 现存版本 | 目标版本
---|---|---|---|---
k8s-master1 | 192.168.1.105 | master1 etcd1 | v1.19.7 | v1.20.4
k8s-master2 | 192.168.1.106 | master1 etcd1 | v1.19.7 | v1.20.4
k8s-master3 | 192.168.1.107 | master1 etcd1 | v1.19.7 | v1.20.4
k8s-node1 | 192.168.1.120 | node1 | v1.19.7 | v1.20.4
k8s-node2 | 192.168.1.121 | node2 | v1.19.7 | v1.20.4
k8s-node3 | 192.168.1.122 | node3 | v1.19.7 | v1.20.4

### 升级预检
详细见[【集群升级-理论指南】](https://mp.weixin.qq.com/s/JsQIsCIA2tqChmA4T098Bw)

---

### 升级主控制平面节点
**节点升级顺序**
-  升级主控制平面节点
-  升级其他控制平面节点
-  升级工作节点
#### 登录k8s-master1
```
ssh root@k8s-master1
```
#### 版本查询
```
#查询目标版本是否存在
yum list --showduplicates kubeadm --disableexcludes=kubernetes|grep 1.20.4

#不存在时，执行
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

#### 腾空节点
在对kubelet作次版本升级时需要腾空节点。对于控制面节点，其上可能运行着CoreDNS Pods或者其它非常重要的负载

**k8s-master1**标记节点为不可调度状态，并驱逐节点上面的pod
```
kubectl cordon k8s-master1
kubectl drain k8s-master1 --delete-local-data --ignore-daemonsets --force
# --ignore-daemonsets 无视DaemonSet管理下的Pod
# --force 当pod不是由ReplicationController,ReplicaSet,Job, DaemonSet或StatefulSet管理时强制执行
# --delete-local-data 强制删除emptyDir类型的volume。emptyDir类型的volume在pod分配到node上时被创建，kubernetes会在node上自动分配 一个目录，因此无需指定宿主机node上对应的目录文件。这个目录的初始内容为空，当Pod从node上移除时，emptyDir中的数据会被永久删除。
```
升级后，因为容器规约的哈希值已更改，所有容器都会被重新启动。

#### 升级kubeadm
```
yum install -y kubeadm-1.20.4-0 --disableexcludes=kubernetes

# 验证版本：kubeadm version
```

#### 验证升级计划
此命令检查你的集群是否可被升级，并获得你要升级的目标版本。 命令也会显示一个包含组件配置版本状态的表格。
```
kubeadm upgrade plan
```
**注意**
- `kubeadm upgrade`也会自动对 kubeadm 在节点上所管理的证书执行续约操作。 如果需要略过证书续约操作，可以使用标志`--certificate-renewal=false`。 更多的信息，可参阅[证书管理指南](https://kubernetes.io/zh/docs/tasks/administer-cluster/kubeadm/kubeadm-certs)。
- `kubeadm upgrade plan` 给出任何需要手动升级的组件配置，用户必须 通过 `--config`命令行标志向`kubeadm upgrade apply`命令提供替代的配置文件。 如果不这样做，`kubeadm upgrade apply`会出错并退出，不再执行升级操作。

#### 准备目标镜像
**查询升级所需镜像**
```
[root@k8s-master1 ~]# kubeadm config images list
k8s.gcr.io/kube-apiserver:v1.20.4
k8s.gcr.io/kube-controller-manager:v1.20.4
k8s.gcr.io/kube-scheduler:v1.20.4
k8s.gcr.io/kube-proxy:v1.20.4
k8s.gcr.io/pause:3.2
k8s.gcr.io/etcd:3.4.13-0
k8s.gcr.io/coredns:1.7.0
```

**查询阿里云仓库中是否存在所需镜像，比对版本**
```
[root@k8s-master1 ~]# kubeadm config images list   --image-repository registry.aliyuncs.com/google_containers
registry.aliyuncs.com/google_containers/kube-apiserver:v1.20.4
registry.aliyuncs.com/google_containers/kube-controller-manager:v1.20.4
registry.aliyuncs.com/google_containers/kube-scheduler:v1.20.4
registry.aliyuncs.com/google_containers/kube-proxy:v1.20.4
registry.aliyuncs.com/google_containers/pause:3.2
registry.aliyuncs.com/google_containers/etcd:3.4.13-0
registry.aliyuncs.com/google_containers/coredns:1.7.0
```

**拉取所需镜像**
```
kubeadm config images pull    --image-repository registry.aliyuncs.com/google_containers
```

**修改镜像**
```
docker tag registry.aliyuncs.com/google_containers/kube-apiserver:v1.20.4 k8s.gcr.io/kube-apiserver:v1.20.4
docker tag registry.aliyuncs.com/google_containers/kube-controller-manager:v1.20.4 k8s.gcr.io/kube-controller-manager:v1.20.4
docker tag registry.aliyuncs.com/google_containers/kube-scheduler:v1.20.4 k8s.gcr.io/kube-scheduler:v1.20.4
docker tag registry.aliyuncs.com/google_containers/kube-proxy:v1.20.4 k8s.gcr.io/kube-proxy:v1.20.4
docker tag registry.aliyuncs.com/google_containers/pause:3.2 k8s.gcr.io/pause:3.2
docker tag registry.aliyuncs.com/google_containers/etcd:3.4.13-0 k8s.gcr.io/etcd:3.4.13-0
docker tag registry.aliyuncs.com/google_containers/coredns:1.7.0 k8s.gcr.io/coredns:1.7.0
```

#### 升级目标版本
```
kubeadm upgrade apply v1.20.4
```
升级成功
```
[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.20.x". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```
#### 升级非核心插件
升级前，你应该已经准备好了升级脚本。比如：
- 手动升级你的 CNI 驱动插件。你的容器网络接口（CNI）驱动应该提供了程序自身的升级说明。 参阅[插件](https://kubernetes.io/zh/docs/concepts/cluster-administration/addons/)页面查找你的CNI驱动， 并查看是否需要其他升级步骤。若**CNI驱动作为DaemonSet运行，则在其他控制平面节点上不需要此步骤**。

#### 升级kubelet和kubectl，并重启服务
```
yum install -y kubelet-1.20.4-0 kubectl-1.20.4-0 --disableexcludes=kubernetes

systemctl daemon-reload
systemctl restart kubelet
```

#### 标记节点为可调度状态
```
kubectl uncordon k8s-master1
```

---

### 对于其它控制面节点

#### 登录其它控制面节点
```
ssh root@k8s-master2
```

#### 设置节点为不可调度状态，并驱逐pod
```
kubectl cordon k8s-master2
kubectl drain k8s-master2 --delete-local-data --ignore-daemonsets --force
```
#### 安装kubeadm
```
yum install -y kubeadm-1.20.4-0 --disableexcludes=kubernetes
```

#### 拉取并修改镜像
```
kubeadm config images pull    --image-repository registry.aliyuncs.com/google_containers

docker tag registry.aliyuncs.com/google_containers/kube-apiserver:v1.20.4 k8s.gcr.io/kube-apiserver:v1.20.4
docker tag registry.aliyuncs.com/google_containers/kube-controller-manager:v1.20.4 k8s.gcr.io/kube-controller-manager:v1.20.4
docker tag registry.aliyuncs.com/google_containers/kube-scheduler:v1.20.4 k8s.gcr.io/kube-scheduler:v1.20.4
docker tag registry.aliyuncs.com/google_containers/kube-proxy:v1.20.4 k8s.gcr.io/kube-proxy:v1.20.4
docker tag registry.aliyuncs.com/google_containers/pause:3.2 k8s.gcr.io/pause:3.2
docker tag registry.aliyuncs.com/google_containers/etcd:3.4.13-0 k8s.gcr.io/etcd:3.4.13-0
docker tag registry.aliyuncs.com/google_containers/coredns:1.7.0 k8s.gcr.io/coredns:1.7.0
```
#### 更新目标版本
```
kubeadm upgrade node

# 其他控制平面节点不需要验证升级计划：kubeadm upgrade plan
# 注意：第一个升级的主控制平面执行的是：kubeadm upgrade apply
# 若CNI驱动作为DaemonSet运行，则在其他控制平面节点上不需要此步骤。
```

#### 升级kubelet和kubectl，并重载服务
```
yum install -y kubelet-1.20.4-0 kubectl-1.20.4-0 --disableexcludes=kubernetes

systemctl daemon-reload
systemctl restart kubelet
```

#### 标记节点为可调度状态
```
kubectl uncordon k8s-master2
```

---

### 升级工作节点
工作节点上的升级过程应该一次执行一个节点，或者一次执行几个节点， 以不影响运行工作负载所需的最小容量。

#### 登录节点
```
ssh root@k8s-node1
```

#### 设置节点为不可调度状态，并驱逐pod
```
kubectl cordon k8s-node1
kubectl drain k8s-node1 --delete-local-data --ignore-daemonsets --force
```

#### 升级kubeadm
```
yum install -y kubeadm-1.20.4-0 --disableexcludes=kubernetes
```

#### 拉取并修改镜像
```
kubeadm config images pull    --image-repository registry.aliyuncs.com/google_containers

docker tag registry.aliyuncs.com/google_containers/kube-apiserver:v1.20.4 k8s.gcr.io/kube-apiserver:v1.20.4
docker tag registry.aliyuncs.com/google_containers/kube-controller-manager:v1.20.4 k8s.gcr.io/kube-controller-manager:v1.20.4
docker tag registry.aliyuncs.com/google_containers/kube-scheduler:v1.20.4 k8s.gcr.io/kube-scheduler:v1.20.4
docker tag registry.aliyuncs.com/google_containers/kube-proxy:v1.20.4 k8s.gcr.io/kube-proxy:v1.20.4
docker tag registry.aliyuncs.com/google_containers/pause:3.2 k8s.gcr.io/pause:3.2
docker tag registry.aliyuncs.com/google_containers/etcd:3.4.13-0 k8s.gcr.io/etcd:3.4.13-0
docker tag registry.aliyuncs.com/google_containers/coredns:1.7.0 k8s.gcr.io/coredns:1.7.0
```

#### 升级本地的 kubelet 配置
```
kubeadm upgrade node
```

#### 升级kubelet和kubectl，并重载服务
```
yum install -y kubelet-1.20.4-0 kubectl-1.20.4-0 --disableexcludes=kubernetes

systemctl daemon-reload
systemctl restart kubelet
```

#### 标记节点为可调度状态
```
kubectl uncordon k8s-node1
```

#### 检查
```
# 检查节点状态
[root@k8s-master1 ~]# kubectl get nodes
NAME          STATUS   ROLES                  AGE   VERSION
k8s-master1   Ready    control-plane,master   12d   v1.20.4
k8s-master2   Ready    control-plane,master   11d   v1.20.4
k8s-master3   Ready    control-plane,master   11d   v1.20.4
k8s-node1     Ready    <none>                 11d   v1.20.4
k8s-node2     Ready    <none>                 11d   v1.20.4
k8s-node3     Ready    <none>                 11d   v1.20.4

# 监控各节点日志
tail -f /var/log/messages

# 监控apiserver日志 
kubectl logs -f -n kube-system kube-apiserver-k8s-master2
```

---

### 从故障状态恢复
若`kubeadm upgrade`失败并且没有回滚，如由于执行期间节点意外关闭， 你可以再次运行`kubeadm upgrade`。此命令是幂等的，并最终确保实际状态是你声明的期望状态。 要从故障状态恢复，你还可以运行`kubeadm upgrade --force`而无需更改集群正在运行的版本。

在升级期间，kubeadm向`/etc/kubernetes/tmp`目录下的如下备份文件夹写入数据：
- kubeadm-backup-etcd-<date>-<time>
- kubeadm-backup-manifests-<date>-<time>

`kubeadm-backup-etcd`包含当前控制面节点本地etcd成员数据的备份。若etcd 升级失败并且自动回滚也无法修复，则可以将此文件夹中的内容复制到`/var/lib/etcd`进行手工修复。如果使用的是外部的etcd，则此备份文件夹为空。

`kubeadm-backup-manifests`包含当前控制面节点的静态Pod 清单文件的备份版本。 若升级失败并且无法自动回滚，则此文件夹中的内容可以复制到`/etc/kubernetes/manifests`目录实现手工恢复。 若由于某些原因，在升级前后某个组件的清单未发生变化，则kubeadm也不会为之生成备份版本。

---

### 工作原理
kubeadm upgrade apply 做了以下工作：

- 检查你的集群是否处于可升级状态:
1. API 服务器是可访问的
2. 所有节点处于 Ready 状态
3. 控制面是健康的
- 强制执行版本偏差策略
- 确保控制面的镜像是可用的或可拉取到服务器上
- 如果组件配置要求版本升级，则生成替代配置与/或使用用户提供的覆盖版本配置
- 升级控制面组件或回滚（如果其中任何一个组件无法启动）
- 应用新的 kube-dns 和 kube-proxy 清单，并强制创建所有必需的 RBAC 规则
- 如果旧文件在 180 天后过期，将创建 API 服务器的新证书和密钥文件并备份旧文件

**kubeadm upgrade node**在其他控制平节点上执行以下操作：
- 从集群中获取 kubeadm ClusterConfiguration
- （可选操作）备份 kube-apiserver 证书
- 升级控制平面组件的静态 Pod 清单
- 为本节点升级 kubelet 配置

kubeadm upgrade node 在工作节点上完成以下工作：
- 从集群取回 kubeadm ClusterConfiguration。
- 为本节点升级 kubelet 配置。
