## <center>前情提要

#### 实现目的
部署一个高可用的Kubernetes集群。高可用指apiserver、scheduler、controller-manager、tcd等组件的多副本实现以及负载均衡。

#### 版本信息
本文档主要组件的版本信息：
- 操作系统：centos7
- docker：19.03.9
- kubelet：v1.19.7
- kubectl：v1.19.7
- golang：go1.15.5
- etcd：3.4.9
- openresty：1.19.3.1

#### 硬件要求
- master节点和node节点硬件配置至少CPU双核，内存4G，磁盘空闲50G
- 每个主机的主机名必须唯一
- 每个主机的MAC地址必须唯一
- 每个主机的product_uuid(cat /sys/class/dmi/id/product_uuid)必须唯一

#### 端口开放：
- master节点
```
端口范围			用途
6443 *			Kubernetes API server
2379-2380		etcd server client API
10250			Kubelet API
10251			kube-scheduler
10252			kube-controller-manager
10255			Read-only Kubelet AP(Heapster)
```
- node节点
```
端口范围			用途
10250			Kubelet API
10255			Read-only Kubelet API (Heapster)
30000-32767		NodePort Services默认端口范围。
```

#### 系统配置（master节点和node节点）

**关闭selinux**
```
sed -i 's/\=enforcing/\=disabled/' /etc/selinux/config 
setenforce 0
```

**关闭swap**
```
vim /etc/fstab		#注释swap的挂载
mount -a
swapoff -a
```

**关闭防火墙**
```
service firewalld stop
chkconfig firewalld off
```

**时间同步**
```
yum -y install ntpdate
echo "0 12 * * * /usr/sbin/ntpdate cn.pool.ntp.org" >> /etc/crontab
service crond restart
chkconfig crond on
```

**将桥接的IPv4流量传递到iptables的链**
```
cat > /etc/sysctl.d/k8s.conf << EOF
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1

net.ipv4.ip_forward = 1

#iptables透明网桥的实现
# NOTE: kube-proxy 要求 NODE 节点操作系统中要具备 /sys/module/br_netfilter 文件，而且还要设置 bridge-nf-call-iptables=1，如果不满足要求，那么 kube-proxy 只是将检查信息记录到日志中，kube-proxy 仍然会正常运行，但是这样通过 Kube-proxy 设置的某些 iptables 规则就不会工作。

net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-arptables = 1
EOF
sysctl --system
```

**命令自动补全工具**
```
yum -y install  bash-completion && \
chmod a+x /usr/share/bash-completion/bash_completion && \
/usr/share/bash-completion/bash_completion && \
exit  #配置完成后重启终端
```

**具体可看官档**：
- master节点配置要求：[最低要求](https://v1-19.docs.kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin)
- node节点配置要求：[最低要求](https://v1-19.docs.kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin)


#### 节点分布信息，写入所有master节点和node节点的/etc/hosts
```
127.0.0.1 k8s-apiserver-entrypoint
192.168.1.105 k8s-master1 etcd1 apiserver1
192.168.1.106 k8s-master2 etcd2 apiserver2
192.168.1.107 k8s-master3 etcd3 apiserver3
192.168.1.120 k8s-node1
192.168.1.121 k8s-node2
192.168.1.122 k8s-node3
192.168.1.123 k8s-node4
192.168.1.124 k8s-node5
```
#### 部署架构图
![](https://github.com/erbiao3k/self-doc/blob/master/kubernetes/pic/arch.png?raw=true)

**说明**：

**apiserver负载均衡**：
- 在每个节点部署一个openresty，负载均衡去往apiserver的请求到不同的kube-apiserver，借此实现apiserver的高可用。
- 此种做法需要修改集群整体配置，以求将去往apiserver的请求全部从openresty进入

---

# <center>组件部署

#### 部署顺序
- **控制平面master节点**
0. 系统配置
1. 部署etcd集群
2. 部署docker
3. 部署kubelctl、kubelet、kubeadm
4. 准备集群镜像
5. 初始化集群
6. 增加控制平面master节点
7. 部署apiserver负载均衡入口openresty
8. 调整集群apiserver配置
- **工作平面node节点**
0. 系统配置
1. 部署docker
1. 部署apiserver负载均衡入口openresty
2. 安装kubelctl、kubelet、kubeadm
3. 准备集群镜像
4. 增加工作平面node节点

## 控制平面master节点
0. 系统配置
1. 部署etcd集群
2. 部署docker
3. 部署kubelctl、kubelet、kubeadm
4. 准备集群镜像
5. 初始化集群
6. 增加控制平面master节点
7. 部署apiserver负载均衡入口openresty
8. 调整集群apiserver配置
### 部署etcd集群

**部署阿里云仓库源信息**
```
cat << 'EOF' > /etc/yum.repos.d/base.repo
[base]
name=CentOS-$releasever - Base - mirrors.aliyun.com
baseurl=http://mirrors.aliyun.com/centos/$releasever/os/$basearch/
gpgcheck=0

[updates]
name=CentOS-$releasever - Updates - mirrors.aliyun.com
baseurl=http://mirrors.aliyun.com/centos/$releasever/updates/$basearch/
gpgcheck=0

[extras]
name=CentOS-$releasever - Extras - mirrors.aliyun.com
baseurl=http://mirrors.aliyun.com/centos/$releasever/extras/$basearch/
gpgcheck=0
EOF

cat << 'EOF' > /etc/yum.repos.d/epel.repo
[epel]
name=Extra Packages for Enterprise Linux 7 - $basearch
baseurl=http://mirrors.aliyun.com/epel/7/$basearch
enabled=1
gpgcheck=0
EOF
yum clean all && yum makecache
```


**在k8s-master1安装etcdadm**

使用etcdadm方式进行部署：

优点：一键搞定证书 + etcd + 扩容问题；

缺点：支持的 etcd 启动参数非常少，安装后需要通过 etcd.env 调整运行参数。

```
#安装git
yum -y install git go

#下载源码包
cd /usr/local
git clone https://github.com/kubernetes-sigs/etcdadm.git
cd etcdadm

export GOPROXY=https://goproxy.io

make etcdadm

mv ./etcdadm /usr/bin/etcdadm 
```
编译过程中若出现“fatal: git fetch-pack: expected shallow list”错误，可能是golang或git版本过低。建议升级golang或将git版本升级到2.x。
centos升级git到2.x：[地址](https://github.com/erbiao3k/self-doc/blob/master/Linux/centos%E5%8D%87%E7%BA%A7git%E5%88%B02.x.md)

**在k8s-master1启动etcd服务(etcd1)**

初始化过程中出现错误执行`etcdadm reset`后重试。

```
etcdadm init --certs-dir /etc/kubernetes/pki/etcd/ --install-dir /usr/bin/
#--certs-dir     生成的证书存放位置
#--install-dir   指令存放位置
```

启动过程：
```
INFO[0000] [install] Removing existing data dir "/var/lib/etcd" 
INFO[0000] [install] Artifact not found in cache. Trying to fetch from upstream: https://github.com/coreos/etcd/releases/download 
INFO[0000] [install] Downloading & installing etcd https://github.com/coreos/etcd/releases/download from 3.4.9 to /var/cache/etcdadm/etcd/v3.4.9 
INFO[0000] [install] downloading etcd from https://github.com/coreos/etcd/releases/download/v3.4.9/etcd-v3.4.9-linux-amd64.tar.gz to /var/cache/etcdadm/etcd/v3.4.9/etcd-v3.4.9-linux-amd64.tar.gz 
######################################################################## 100.0%
INFO[0008] [install] extracting etcd archive /var/cache/etcdadm/etcd/v3.4.9/etcd-v3.4.9-linux-amd64.tar.gz to /tmp/etcd474716663 
INFO[0009] [install] verifying etcd 3.4.9 is installed in /usr/bin/ 
INFO[0009] [certificates] creating PKI assets           
INFO[0009] creating a self signed etcd CA certificate and key files 
[certificates] Generated ca certificate and key.
INFO[0009] creating a new server certificate and key files for etcd 
[certificates] Generated server certificate and key.
[certificates] server serving cert is signed for DNS names [k8s-master1] and IPs [192.168.1.105 127.0.0.1]
INFO[0009] creating a new certificate and key files for etcd peering 
[certificates] Generated peer certificate and key.
[certificates] peer serving cert is signed for DNS names [k8s-master1] and IPs [192.168.1.105]
INFO[0009] creating a new client certificate for the etcdctl 
[certificates] Generated etcdctl-etcd-client certificate and key.
INFO[0009] creating a new client certificate for the apiserver calling etcd 
[certificates] Generated apiserver-etcd-client certificate and key.
[certificates] valid certificates and keys now exist in "/etc/kubernetes/pki/etcd/"
INFO[0010] [health] Checking local etcd endpoint health 
INFO[0010] [health] Local etcd endpoint is healthy      
INFO[0010] To add another member to the cluster, copy the CA cert/key to its certificate dir and run: 
INFO[0010] 	etcdadm join https://192.168.1.105:2379  
```

查看etcd服务状态为已启动状态
```
systemctl status etcd.service

```

**复制证书、etcdadm到etcd2(k8s-master2)，etcd2(k8s-master3)**
```
# etcd2(k8s-master2)，etcd2(k8s-master3)执行
mkdir /etc/kubernetes/pki/etcd/ -p 

# 在etcd1(k8s-master1)复制证书
scp /etc/kubernetes/pki/etcd/ca.* etcd2:/etc/kubernetes/pki/etcd/
scp /etc/kubernetes/pki/etcd/ca.* etcd3:/etc/kubernetes/pki/etcd/

# 在etcd1(k8s-master1)复制etcdadm
scp /usr/bin/etcdadm etcd2:/usr/bin/
scp /usr/bin/etcdadm etcd3:/usr/bin/
```

**在etcd2(k8s-master2)、etcd3(k8s-master3)启动etcd服务，并加入etcd集群**

初始化过程中出现错误执行`etcdadm reset`后重试。

```
etcdadm join https://192.168.1.105:2379 --certs-dir /etc/kubernetes/pki/etcd/ --install-dir /usr/bin/

```

**k8s-master1、k8s-master2、k8s-master3配置etcdctl快捷指令**
```
echo -e "export ETCDCTL_API=3\nalias etcdctl='etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key --endpoints=https://192.168.1.105:2379,https://192.168.1.106:2379,https://192.168.1.107:2379 --write-out=table'" >> /root/.bashrc; source /root/.bashrc


```

**检查etcd集群状态**
```
etcdctl endpoint health

+----------------------------+--------+-------------+-------+
|          ENDPOINT          | HEALTH |    TOOK     | ERROR |
+----------------------------+--------+-------------+-------+
| https://192.168.1.105:2379 |   true | 15.241521ms |       |
| https://192.168.1.107:2379 |   true | 21.073974ms |       |
| https://192.168.1.106:2379 |   true | 22.514348ms |       |
+----------------------------+--------+-------------+-------+

查看etcd服务启动日志，可通过tail -f /var/log/message动态查看
```

---

### 部署docker
**master节点和node节点安装docker**
```
yum -y remove docker*
yum install -y yum-utils device-mapper-persistent-data lvm2 && \
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo && \
yum makecache fast && \
yum install   -y  docker-ce-3:19.03.9-3.el7.x86_64 docker-ce-cli-1:19.03.9-3.el7.x86_64 && \
chkconfig docker on

#master节点和node节点配置docker国内镜像加速
mkdir /etc/docker
cat > /etc/docker/daemon.json << EOF
{
    "exec-opts": ["native.cgroupdriver=systemd"],
    "registry-mirrors": [
        "https://1nj0zren.mirror.aliyuncs.com",
        "https://kfwkfulq.mirror.aliyuncs.com",
        "https://2lqq34jg.mirror.aliyuncs.com",
        "https://pee6w651.mirror.aliyuncs.com",
        "http://hub-mirror.c.163.com",
        "https://docker.mirrors.ustc.edu.cn",
        "http://f1361db2.m.daocloud.io",
        "https://registry.docker-cn.com"
    ]
}
EOF
systemctl daemon-reload
systemctl restart docker
cat /etc/docker/daemon.json

#yum list docker-ce.x86_64 --showduplicates | sort -r 查询可供安装的历史版本
```

---

### 安装kubelctl、kubelet、kubeadm
**master节点和node节点安装kubelctl、kubelet、kubeadm**
```
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

#在etcd1(k8s-master1)，etcd2(k8s-master2)，etcd3(k8s-master3)分别复制证书

cp /etc/kubernetes/pki/etcd/apiserver-etcd-client.crt /etc/kubernetes/pki/
cp /etc/kubernetes/pki/etcd/apiserver-etcd-client.key /etc/kubernetes/pki/

#此证书只有一年有效期，需注意。
#openssl x509 -in apiserver-etcd-client.crt  -noout -text #查看证书有效期

yum -y install  kubectl-1.19.7-0.x86_64 kubelet-1.19.7-0.x86_64 kubeadm-1.19.7-0.x86_64
systemctl daemon-reload
systemctl enable kubelet


注意，这里不需要启动kubelet，初始化的过程中会自动启动的，如果此时启动了会出现如下报错，忽略即可。日志在tail -f /var/log/messages：
failed to load Kubelet config file /var/lib/kubelet/config.yaml, error failed to read kubelet config file “/var/lib/kubelet/config.yaml”, error: open /var/lib/kubelet/config.yaml: no such file or directory
```

### 准备集群镜像
```
#查询默认镜像仓库的镜像地址信息
kubeadm config images list

k8s.gcr.io/kube-apiserver:v1.19.7
k8s.gcr.io/kube-controller-manager:v1.19.7
k8s.gcr.io/kube-scheduler:v1.19.7
k8s.gcr.io/kube-proxy:v1.19.7
k8s.gcr.io/pause:3.2
k8s.gcr.io/etcd:3.4.13-0 #未使用，可不拉取
k8s.gcr.io/coredns:1.7.0

#查询国内镜像仓库地的镜像地址信息
kubeadm config images list   --image-repository registry.aliyuncs.com/google_containers

registry.aliyuncs.com/google_containers/kube-apiserver:v1.19.7
registry.aliyuncs.com/google_containers/kube-controller-manager:v1.19.7
registry.aliyuncs.com/google_containers/kube-scheduler:v1.19.7
registry.aliyuncs.com/google_containers/kube-proxy:v1.19.7
registry.aliyuncs.com/google_containers/pause:3.2
registry.aliyuncs.com/google_containers/etcd:3.4.13-0 #未使用，可不拉取
registry.aliyuncs.com/google_containers/coredns:1.7.0

#k8s-master1、k8s-master2、k8s-master3拉取国内镜像仓库地的镜像
kubeadm config images pull    --image-repository registry.aliyuncs.com/google_containers

#k8s-master1、k8s-master2、k8s-master3修改镜像tag
docker tag registry.aliyuncs.com/google_containers/kube-apiserver:v1.19.7 k8s.gcr.io/kube-apiserver:v1.19.7
docker tag registry.aliyuncs.com/google_containers/kube-controller-manager:v1.19.7 k8s.gcr.io/kube-controller-manager:v1.19.7
docker tag registry.aliyuncs.com/google_containers/kube-scheduler:v1.19.7 k8s.gcr.io/kube-scheduler:v1.19.7
docker tag registry.aliyuncs.com/google_containers/kube-proxy:v1.19.7 k8s.gcr.io/kube-proxy:v1.19.7
docker tag registry.aliyuncs.com/google_containers/pause:3.2 k8s.gcr.io/pause:3.2
docker tag registry.aliyuncs.com/google_containers/etcd:3.4.13-0 k8s.gcr.io/etcd:3.4.13-0
docker tag registry.aliyuncs.com/google_containers/coredns:1.7.0 k8s.gcr.io/coredns:1.7.0

```

---

### 初始化集群
**在k8s-master1上初始化k8s配置文件**
```
cat > kubeadm-config.yml << EOF
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
nodeRegistration:
  name: $HOSTNAME     #对应节点名称
---
apiServer:
  extraArgs:
    authorization-mode: Node,RBAC
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: k8s-apiserver-entrypoint:6443
controllerManager: {}
dns:
  type: CoreDNS
etcd:         
  external:  # 指定外部 etcd
    caFile: /etc/kubernetes/pki/etcd/ca.crt
    certFile: /etc/kubernetes/pki/apiserver-etcd-client.crt
    endpoints:                        #etcd集群节点
    - https://192.168.1.105:2379
    - https://192.168.1.106:2379
    - https://192.168.1.107:2379
    keyFile: /etc/kubernetes/pki/apiserver-etcd-client.key
#imageRepository: registry.aliyuncs.com/google_containers    #替换成国内的镜像仓库源
imagePullPolicy: IfNotPresent
kind: ClusterConfiguration
kubernetesVersion: v1.19.7 # kubernetes版本
networking:
  dnsDomain: cluster.local
  podSubnet: 100.2.0.0/16    # pod 网段，对应 flannel 插件
  serviceSubnet: 172.16.0.0/16  # svc网段
scheduler: {}
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs    # 修改 kube-proxy 模式
EOF
```

**在k8s-master1初始化控制平面master节点**
```
kubeadm init --config kubeadm-config.yml --upload-certs --v=5
```

初始化完成后，会提示三段信息，要牢记：
第一段用于集群新增控制平面master节点
```
You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join k8s-apiserver-entrypoint:6443 --token 2s5xv9.sfxbm234ko23pb4g \
    --discovery-token-ca-cert-hash sha256:115097f1f47c38d6f2fc58fe8dcafe0ae8d84f320762b0d54af231919528cb53 \
    --control-plane --certificate-key 029844065f6ccb793158028047fac3e86dbd31eb9295daec21e14a9aa47af6eb

```
第二段用于集群新增工作平面node节点
```
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join k8s-apiserver-entrypoint:6443 --token 2s5xv9.sfxbm234ko23pb4g \
    --discovery-token-ca-cert-hash sha256:115097f1f47c38d6f2fc58fe8dcafe0ae8d84f320762b0d54af231919528cb53 
```
第三段用于kubectl工具访问集群信息的配置
```
# 初始化完毕后提示以下三条命令，记得执行！
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

配置kubectl命令自动补全工具
```
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
kubectl version
exit
```

**k8s-master1上初始化集群网络插件flannel**
```
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

pod网络和集群初始化配置保持一致
```
  net-conf.json: |
    {
      "Network": "100.2.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
```

安装集群网络插件
```
kubectl apply -f kube-flannel.yml
```
查看集群pod状态
```
kubectl get pods -A


NAMESPACE     NAME                                  READY   STATUS    RESTARTS   AGE
kube-system   coredns-6d56c8448f-kdvp2              1/1     Running   0          38m
kube-system   coredns-6d56c8448f-xw7lr              1/1     Running   0          38m
kube-system   kube-apiserver-k8s-master1            1/1     Running   0          38m
kube-system   kube-controller-manager-k8s-master1   1/1     Running   0          38m
kube-system   kube-flannel-ds-9w5qw                 1/1     Running   0          80s
kube-system   kube-proxy-6rdl4                      1/1     Running   0          38m
kube-system   kube-scheduler-k8s-master1            1/1     Running   0          38m

```

---

### 增加控制平面master节点
**在k8s-master2，k8s-master3上将节点加入控制平面**

在上述配置中，已将**k8s-apiserver-entrypoint**指向**127.0.0.1**，在k8s-master2，k8s-master3先修改指向为**k8s-master1**的内网IP:**192.168.1.105**
```
192.168.1.105 k8s-apiserver-entrypoint

```

在k8s-master2，k8s-master3将节点加入控制平面
```
kubeadm join k8s-apiserver-entrypoint:6443 --token 2s5xv9.sfxbm234ko23pb4g --discovery-token-ca-cert-hash sha256:115097f1f47c38d6f2fc58fe8dcafe0ae8d84f320762b0d54af231919528cb53 --control-plane --certificate-key 029844065f6ccb793158028047fac3e86dbd31eb9295daec21e14a9aa47af6eb
```

在k8s-master2，k8s-master3初始化完毕后提示以下三条命令，记得执行！
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

检查pod初始化情况
```
[root@k8s-master3 ~]# kubectl get pods -A
NAMESPACE     NAME                                  READY   STATUS    RESTARTS   AGE
kube-system   coredns-6d56c8448f-kdvp2              1/1     Running   0          78m
kube-system   coredns-6d56c8448f-xw7lr              1/1     Running   0          78m
kube-system   kube-apiserver-k8s-master1            1/1     Running   0          78m
kube-system   kube-apiserver-k8s-master2            1/1     Running   0          13m
kube-system   kube-apiserver-k8s-master3            1/1     Running   0          111s
kube-system   kube-controller-manager-k8s-master1   1/1     Running   0          78m
kube-system   kube-controller-manager-k8s-master2   1/1     Running   0          13m
kube-system   kube-controller-manager-k8s-master3   1/1     Running   0          112s
kube-system   kube-flannel-ds-24wl2                 1/1     Running   0          113s
kube-system   kube-flannel-ds-4h2t4                 1/1     Running   0          13m
kube-system   kube-flannel-ds-9w5qw                 1/1     Running   0          41m
kube-system   kube-proxy-6rdl4                      1/1     Running   0          78m
kube-system   kube-proxy-csgx7                      1/1     Running   0          113s
kube-system   kube-proxy-d7x9x                      1/1     Running   0          13m
kube-system   kube-scheduler-k8s-master1            1/1     Running   0          78m
kube-system   kube-scheduler-k8s-master2            1/1     Running   0          13m
kube-system   kube-scheduler-k8s-master3            1/1     Running   0          112s
```

将k8s-master2，k8s-master3上**k8s-apiserver-entrypoint**指向内网IP:**192.168.1.105**修改回**127.0.0.1**
```
127.0.0.1 k8s-apiserver-entrypoint
```

---

### 部署apiserver负载均衡入口openresty
**依据上述架构图，需要在每节点部署nginx以反向代理和负载均衡apiserver的请求**
#### 所有master节点和node节点部署openresty
```
yum install yum-utils -y
yum-config-manager --add-repo https://openresty.org/package/centos/openresty.repo 
yum install openresty -y
mkdir /var/log/openresty
# 写入nginx.conf配置文件
cat > /usr/local/openresty/nginx/conf/nginx.conf << EOF
worker_processes  auto;
error_log  /var/log/openresty/error-ngx.log  notice;

worker_rlimit_nofile 65535;
worker_shutdown_timeout 10s;

events {
    worker_connections  10240;
    use epoll;
    multi_accept on;
}
        
stream {
    log_format main "\$remote_addr  \$upstream_addr  \$time_local \$status";
    upstream kube_apiserver {
        least_conn;
        #apiserver接口
        server 192.168.1.105:6443;
        server 192.168.1.106:6443;
        server 192.168.1.107:6443;
    }

    server {
        listen        0.0.0.0:8888;
        access_log /var/log/openresty/k8s-apiserver-access.log main;
        proxy_pass    kube_apiserver;
        proxy_timeout 10m;
        proxy_connect_timeout 1s;
    }
}
EOF

openresty -t && chkconfig openresty on && service openresty restart

```

### 调整集群apiserver配置
**替换k8s-master1、k8s-master3、k8s-master3负载均衡器地址**
```
# 修改 kubelet 配置
sed -i s'/k8s-apiserver-entrypoint:6443/k8s-apiserver-entrypoint:8888/g'  /etc/kubernetes/kubelet.conf

#修改controller-manager
sed -i s'/k8s-apiserver-entrypoint:6443/k8s-apiserver-entrypoint:8888/g'  /etc/kubernetes/controller-manager.conf

#修改scheduler
sed -i s'/k8s-apiserver-entrypoint:6443/k8s-apiserver-entrypoint:8888/g'  /etc/kubernetes/scheduler.conf

#修改admin.conf
sed -i s'/k8s-apiserver-entrypoint:6443/k8s-apiserver-entrypoint:8888/g'  /etc/kubernetes/admin.conf

#修改kube-proxy配置
kubectl -n kube-system get cm kube-proxy -o yaml > kube-proxy.yaml
sed -i s'/k8s-apiserver-entrypoint:6443/k8s-apiserver-entrypoint:8888/g' kube-proxy.yaml
kubectl apply -f  kube-proxy.yaml

#修改cluster-info配置
kubectl get  -n  kube-public   cm    cluster-info -o yaml > cluster-info.yaml
sed -i s'/k8s-apiserver-entrypoint:6443/k8s-apiserver-entrypoint:8888/g' cluster-info.yaml
kubectl apply -f cluster-info.yaml

#修改kubeadm-config
kubectl get cm -n kube-system kubeadm-config  -o yaml > kubeadm-config.yaml
sed -i s'/k8s-apiserver-entrypoint:6443/k8s-apiserver-entrypoint:8888/g' kubeadm-config.yaml
kubectl apply -f kubeadm-config.yaml

#修改kubectl指令配置
sed -i s'/k8s-apiserver-entrypoint:6443/k8s-apiserver-entrypoint:8888/g' ~/.kube/config
```

**不同kubernetes版本中可能存在不同的配置文件，一般在/var/lib/kublet，/etc/kubernetes**

**依次重启k8s-master1、k8s-master2、k8s-master3。每个master节点重启时，至少保证一个节点的所有组件正常运行**

---

## 工作平面node节点
0. 系统配置
1. 部署docker
1. 部署apiserver负载均衡入口openresty
2. 安装kubelctl、kubelet、kubeadm
3. 准备集群镜像
4. 增加工作平面node节点

### 系统配置
略
### 部署docker
略
### 部署apiserver负载均衡入口openresty
略
### 安装kubelctl、kubelet、kubeadm
证书无需复制，其他略
### 准备集群镜像
略
### 增加工作平面node节点
**由于集群的apiserver信息调整过，因此需要重新生成新增节点的信息**
```
#在任意master节点上执行
kubeadm token create --print-join-command

#输出信息
kubeadm join k8s-apiserver-entrypoint:8888 --token 8qy239.ohw6gp7h3pyq3oj1     --discovery-token-ca-cert-hash sha256:3bb0c1a767ccbf1cb0f64804627d392fa541b52176c8c4b83a0139952c09009f 

```
**在所有node节点上执行**
```
kubeadm join k8s-apiserver-entrypoint:8888 --token 8qy239.ohw6gp7h3pyq3oj1     --discovery-token-ca-cert-hash sha256:3bb0c1a767ccbf1cb0f64804627d392fa541b52176c8c4b83a0139952c09009f 
```

**检查集群节点状态**
```
kubectl get pods -A
kubectl get nodes 
```
