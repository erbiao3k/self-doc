## 部署说明
#### 资源说明
1. **最低配置**
- Master节点：2C 4G 50GB
- Node节点：2C 4GB 50GB

#### 部署说明
1. 为兼容更多场景的部署，所有组件均采用`二进制`部署或源码部署
2. linux内核版本 `≥` `3.8`
3. linux内核必须支持一种适合的存储驱动（storage driver），默认的存储驱动通常是`Device Mapper`或`AUFS`。

#### 集群信息

| 主机名 | 角色 | IP | 组件 |
| --- | --- | --- | --- |
| k8s-master1 | master | 192.168.0.6 | apiserver、controller-manager、etcd、scheduler、docker、nginx |
| k8s-master2 | master | 192.168.0.8 | apiserver、controller-manager、etcd、scheduler、docker、nginx |
| k8s-master3 | master | 192.168.0.15 | apiserver、controller-manager、etcd、scheduler、docker、nginx|
| k8s-node1 | node | 192.168.0.2 | kubelet、kube-proxy、docker、calico、nginx |
| k8s-node2 | node | 192.168.0.13 | kubelet、kube-proxy、docker、calico、nginx|

#### 架构图
![image](https://images.cnblogs.com/cnblogs_com/erbiao/1918053/o_210214061857K8S%E9%AB%98%E5%8F%AF%E7%94%A8%E6%9E%B6%E6%9E%84%E5%9B%BE.png)

#### 组件版本

| 组件 | 版本 |
| --- | --- |
| centos |  7.9 |
| nginx | 1.21.6 |
| kubernetes | 1.23.4 |
| pause | 3.6 |
| coredns | 1.8.6 |
| etcd | 3.5.2 |
| docker | 20.10.8 |
| calico | 3.22.1 |


## 初始化环境

#### /etc/hosts(所有节点操作)
```
cat >> /etc/hosts << EOF
192.168.0.15 k8s-master3
192.168.0.8 k8s-master2
192.168.0.6 k8s-master1
192.168.0.13 k8s-node2
192.168.0.2 k8s-node1
EOF
```

#### 设置主机名(每个对应节点操作)
```
hostnamectl set-hostname  k8s-master1
hostnamectl set-hostname  k8s-master2
hostnamectl set-hostname  k8s-master3
hostnamectl set-hostname  k8s-node1
hostnamectl set-hostname  k8s-node2
```

#### 关闭selinux(所有节点操作)
```
sed -i 's/\=enforcing/\=disabled/' /etc/selinux/config 
setenforce 0
```

#### 关闭swap(所有节点操作)
```
grep -v swap /etc/fstab  > /tmp/fstab
cat /tmp/fstab >/etc/fstab

mount -a
swapoff -a
```

#### 关闭防火墙(所有节点操作)
```
service firewalld stop
chkconfig firewalld off
```

#### 时间同步(所有节点操作)
```
yum -y install chrony*

cat > /etc/chrony.conf <<EOF
pool ntp1.aliyun.com iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
keyfile /etc/chrony.keys
leapsectz right/UTC
logdir /var/log/chrony
EOF

systemctl enable --now chronyd
systemctl restart chronyd
```

#### 内核网络优化(所有节点操作)
```
# 将桥接的IPv4流量传递到iptables的链
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

#### 安装基础软件包(所有节点操作)
```
yum install -y yum-utils device-mapper-persistent-data lvm2 rpcbind device-mapper conntrack socat telnet lsof wget vim make gcc gcc-c++ pcre* ipvsadm net-tools libnl libnl-devel openssl openssl-devel
```

#### 安装iptables(所有节点操作)
```
yum install iptables-services -y

# 关闭即可，k8s自己初始化
service iptables stop   && systemctl disable iptables

# 清空规则
iptables -F
```

#### 开启ipvs(所有节点操作)
```
cat > /etc/sysconfig/modules/ipvs.modules << EOF
modprobe -- ip_vs
modprobe -- ip_vs_lc
modprobe -- ip_vs_wlc
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_lblc
modprobe -- ip_vs_lblcr
modprobe -- ip_vs_dh
modprobe -- ip_vs_sh
modprobe -- ip_vs_nq
modprobe -- ip_vs_sed
modprobe -- ip_vs_ftp
modprobe -- nf_conntrack
EOF

chmod 755 /etc/sysconfig/modules/ipvs.modules
bash /etc/sysconfig/modules/ipvs.modules

lsmod | grep  ip_vs
```

#### 安装docker(在所有节点操作)

下载并部署docker二进制文件
```
wget -P /opt/tempData https://download.docker.com/linux/static/stable/x86_64/docker-20.10.8.tgz
tar xf /opt/tempData/docker-20.10.8.tgz -C /opt/tempData
cp /opt/tempData/docker/* /usr/local/bin
```

生成docker启动配置
```
# 获取当前文件系统最大的分区作为docker服务的数据目录
dockerDisk=`df -Tk|grep -Ev "devtmpfs|tmpfs|overlay"|grep -E "ext4|ext3|xfs"|awk '/\//{print $5,$NF}'|sort -nr|awk '{print $2}'|head -1|tr '\n' ' '|awk '{print $1}'`
dockerDataDir=`echo ${dockerDisk}/dockerData|sed s'/\/\//\//g'`
mkdir ${dockerDataDir}


cat > /etc/systemd/system/docker.service << EOF
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/local/bin/dockerd -H unix:///var/run/docker.sock -H tcp://0.0.0.0:2375 --data-root=${dockerDataDir} --config-file=/opt/docker/daemon.json
ExecReload=/bin/kill -s HUP \$MAINPID
TimeoutSec=0
RestartSec=2
Restart=always

StartLimitBurst=3

LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity

TasksMax=infinity

Delegate=yes

KillMode=process

[Install]
WantedBy=multi-user.target
EOF
```

服务配置文件:
```
mkdir /opt/docker
cat > /opt/docker/daemon.json << EOF
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
```

启动服务
```
systemctl daemon-reload
systemctl enable docker
systemctl restart docker
```

#### 命令自动补全工具(所有节点操作)
```
yum -y install  bash-completion && \
chmod a+x /usr/share/bash-completion/bash_completion && \
/usr/share/bash-completion/bash_completion && \
exit  #配置完成后重启终端
```

## 创建CA(k8s-master1操作)
下载证书生成工具：`cfssl`,`cfssljson`,`cfssl-certinfo`
```
wget -o /usr/local/bin/cfssl https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssl_1.6.1_linux_amd64
wget -o /usr/local/bin/cfssl-certinfo https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssl-certinfo_1.6.1_linux_amd64
wget -o /usr/local/bin/cfssljson https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssljson_1.6.1_linux_amd64

chmod +x /usr/local/bin/*
```

创建文件：`ca-config.json`,`ca-csr.json`
```
mkdir /opt/caCenter -p
# ca配置，876000h为100年
cat > /opt/caCenter/ca-config.json << EOF
{
    "signing": {
        "default": {
            "expiry": "876000h"
        },
        "profiles": {
            "kubernetes": {
                "expiry": "876000h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
EOF

cat > /opt/caCenter/ca-csr.json << EOF
{
  "CN": "kubernetes",
  "key": {
      "algo": "rsa",
      "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "k8s",
      "OU": "system"
    }
  ],
  "ca": {
          "expiry": "876000h"
  }
}
EOF
```

生成CA证书
```
cd /opt/caCenter
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
```


## 部署三节点Etcd集群
#### 生成Etcd证书(k8s-master1操作)
```
# ca证书颁发后，可使用的hosts
mkdir /opt/etcd/{ssl,data} -p
cat > /opt/etcd/ssl/etcd-csr.json << EOF
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "192.168.0.15",
    "192.168.0.8",
    "192.168.0.6"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [{
    "C": "CN",
    "ST": "Beijing",
    "L": "Beijing",
    "O": "k8s",
    "OU": "system"
  }]
}
EOF
```

生成证书
```
cd /opt/etcd/ssl
cfssl gencert -ca=/opt/caCenter/ca.pem -ca-key=/opt/caCenter/ca-key.pem -config=/opt/caCenter/ca-config.json -profile=kubernetes etcd-csr.json | cfssljson  -bare etcd
```


#### 部署第一个Etcd节点(k8s-master1操作)
```
# 下载并解压二进制包
wget -P /opt/tempData  https://github.com/etcd-io/etcd/releases/download/v3.5.2/etcd-v3.5.2-linux-amd64.tar.gz
tar -zxf /opt/tempData/etcd-v3.5.2-linux-amd64.tar.gz -C /opt/tempData
mv /opt/tempData/etcd-v3.5.2-linux-amd64/{etcdctl,etcdutl,etcd} /usr/local/bin/
```

生成脚本`/opt/tempData/etcd.sh`以便配置etcd服务
```
cat > /opt/tempData/etcd.sh << EOF_TOTAL
#!/bin/bash

ETCD_NAME=\$1
ETCD_IP=\$2
ETCD_CLUSTER=\$3

WORK_DIR=/opt/etcd

cat <<EOF_SERVICE >/usr/lib/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/etcd \
--name=\${ETCD_NAME} \
--enable-v2=true \
--data-dir=\${WORK_DIR}/data/default.etcd \
--listen-peer-urls=https://\${ETCD_IP}:2380 \
--listen-client-urls=https://\${ETCD_IP}:2379,http://127.0.0.1:2379 \
--advertise-client-urls=https://\${ETCD_IP}:2379 \
--initial-advertise-peer-urls=https://\${ETCD_IP}:2380 \
--initial-cluster=\${ETCD_NAME}=https://\${ETCD_IP}:2380,\${ETCD_CLUSTER} \
--initial-cluster-token=etcd-cluster \
--initial-cluster-state=new \
--cert-file=\${WORK_DIR}/ssl/etcd.pem \
--key-file=\${WORK_DIR}/ssl/etcd-key.pem \
--peer-cert-file=\${WORK_DIR}/ssl/etcd.pem \
--peer-key-file=\${WORK_DIR}/ssl/etcd-key.pem \
--trusted-ca-file=/opt/caCenter/ca.pem \
--peer-trusted-ca-file=/opt/caCenter/ca.pem
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF_SERVICE

systemctl daemon-reload
systemctl enable etcd
systemctl restart etcd
EOF_TOTAL
```

执行`/opt/tempData/etcd.sh`配置Etcd服务
```
sh /opt/tempData/etcd.sh etcd01 192.168.0.6 etcd02=https://192.168.0.8:2380,etcd03=https://192.168.0.15:2380
```

以上指令中：
- "etcd01"：ETCD_NAME，当前etcd节点名称
- "192.168.0.6"：ETCD_IP，当前etcd节点IP
- "etcd02=https://192.168.0.8:2380,etcd03=https://192.168.0.15:2380"：集群除了当前节点外的连接信息

#### 部署第二个Etcd节点(在k8s-master1操作)
```
# 从k8s-master1节点复制配置信息
scp -r /opt/{caCenter,etcd,tempData} k8s-master2:/opt/
scp -r /usr/local/bin/* k8s-master2:/usr/local/bin/
ssh k8s-master2 "rm -rf /opt/etcd/data"

# 配置并启动etcd服务
ssh k8s-master2 "sh /opt/tempData/etcd.sh etcd02 192.168.0.8 etcd01=https://192.168.0.6:2380,etcd03=https://192.168.0.15:2380"
```

#### 部署第三个Etcd节点(k8s-master1操作)
```
# 从k8s-master1节点复制配置信息
scp -r /opt/{caCenter,etcd,tempData}  k8s-master3:/opt/
scp -r /usr/local/bin/* k8s-master3:/usr/local/bin/
ssh k8s-master3 "rm -rf /opt/etcd/data"

# 配置并启动etcd服务
ssh k8s-master3 "sh /opt/tempData/etcd.sh etcd03 192.168.0.15 etcd01=https://192.168.0.6:2380,etcd02=https://192.168.0.8:2380"
```

#### 配置指令别名(所有etcd服务节点操作)
```
echo "alias etcdctl3='ETCDCTL_API=3 etcdctl --cacert=/opt/caCenter/ca.pem --cert=/opt/etcd/ssl/etcd.pem --key=/opt/etcd/ssl/etcd-key.pem --endpoints="https://192.168.0.6:2379,https://192.168.0.8:2379,https://192.168.0.15:2379"'" >> ~/.bashrc

echo "alias etcdctl2='ETCDCTL_API=2 etcdctl --ca-file=/opt/caCenter/ca.pem --cert-file=/opt/etcd/ssl/etcd.pem --key-file=/opt/etcd/ssl/etcd-key.pem --endpoints="https://192.168.0.6:2379,https://192.168.0.8:2379,https://192.168.0.15:2379"'" >> ~/.bashrc
source ~/.bashrc
```

#### 测试Etcd集群状态(任一Etcd集群节点操作)

```
# 查看集群endpoint健康情况: etcdctl3 endpoint health -w table

+---------------------------+--------+-------------+-------+
|         ENDPOINT          | HEALTH |    TOOK     | ERROR |
+---------------------------+--------+-------------+-------+
|  https://192.168.0.8:2379 |   true | 10.449213ms |       |
|  https://192.168.0.6:2379 |   true |  10.60263ms |       |
| https://192.168.0.15:2379 |   true | 10.666955ms |       |
+---------------------------+--------+-------------+-------+

# 查看集群endpoint健康情况：etcdctl3 endpoint status -w table
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|         ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|  https://192.168.0.6:2379 | f6f06406c5914390 |   3.5.2 |   20 kB |     false |      false |         4 |         78 |                 78 |        |
|  https://192.168.0.8:2379 | cf81cedc69f6ab55 |   3.5.2 |   20 kB |      true |      false |         4 |         78 |                 78 |        |
| https://192.168.0.15:2379 | 5eab3181052a355a |   3.5.2 |   20 kB |     false |      false |         4 |         78 |                 78 |        |
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

## 在Master节点部署组件
```
TLS Bootstrapping 机制

Master apiserver启用TLS认证后，每个节点的 kubelet 组件都要使用由 apiserver 使用的 CA 签发的有效证书才能与 apiserver 通讯，当Node节点很多时，这种客户端证书颁发需要大量工作，同样也会增加集群扩展复杂度。

为了简化流程，Kubernetes引入了TLS bootstraping机制来自动颁发客户端证书，kubelet会以一个低权限用户自动向apiserver申请证书，kubelet的证书由apiserver动态签署。

TLS bootstrapping 具体引导过程

1.TLS 作用
TLS 的作用就是对通讯加密，防止中间人窃听；同时如果证书不信任的话根本就无法与 apiserver 建立连接，更不用提有没有权限向apiserver请求指定内容。

2. RBAC 作用
当 TLS 解决了通讯问题后，那么权限问题就应由 RBAC 解决(可以使用其他权限模型，如 ABAC)；RBAC 中规定了一个用户或者用户组(subject)具有请求哪些 api 的权限；在配合 TLS 加密的时候，实际上 apiserver 读取客户端证书的 CN 字段作为用户名，读取 O字段作为用户组.

以上说明：第一，想要与 apiserver 通讯就必须采用由 apiserver CA 签发的证书，这样才能形成信任关系，建立 TLS 连接；第二，可以通过证书的 CN、O 字段来提供 RBAC 所需的用户与用户组。

kubelet 首次启动流程
TLS bootstrapping 功能是让 kubelet 组件去 apiserver 申请证书，然后用于连接 apiserver；那么第一次启动时没有证书如何连接 apiserver ?

在apiserver 配置中指定了一个 token.csv 文件，该文件中是一个预设的用户配置；同时该用户的Token和由apiserver的CA签发的用户被写入了kubelet 所使用的bootstrap.kubeconfig 配置文件中；这样在首次请求时，kubelet 使用 bootstrap.kubeconfig 中被 apiserver CA 签发证书时信任的用户来与 apiserver 建立 TLS 通讯，使用 bootstrap.kubeconfig 中的用户 Token 来向 apiserver 声明自己的 RBAC 授权身份.token.csv格式:

3940fd7fbb391d1b4d861ad17a1f0613,kubelet-bootstrap,10001,"system:kubelet-bootstrap"
首次启动时，可能与遇到 kubelet 报 401 无权访问 apiserver 的错误；这是因为在默认情况下，kubelet 通过 bootstrap.kubeconfig 中的预设用户 Token 声明了自己的身份，然后创建 CSR 请求；但是不要忘记这个用户在我们不处理的情况下他没任何权限的，包括创建 CSR 请求；所以需要创建一个 ClusterRoleBinding，将预设用户 kubelet-bootstrap 与内置的 ClusterRole system:node-bootstrapper 绑定到一起，使其能够发起 CSR 请求。
```

#### 生成证书(k8s-master1操作)

依据CA证书生成kube-apiserver证书：
```
mkdir /opt/kubernetes/{cfg,ssl} -p
cat >/opt/kubernetes/ssl/kube-apiserver-csr.json << EOF
{
    "CN": "kubernetes",
    "hosts": [
      "10.255.0.1",
      "127.0.0.1",
      "192.168.0.15",
      "192.168.0.13",
      "192.168.0.6",
      "192.168.0.8",
      "192.168.0.2",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF

cd /opt/kubernetes/ssl/
cfssl gencert -ca=/opt/caCenter/ca.pem -ca-key=/opt/caCenter/ca-key.pem -config=/opt/caCenter/ca-config.json -profile=kubernetes kube-apiserver-csr.json | cfssljson -bare kube-apiserver
```

依据CA证书生成kube-proxy证书:
```
cat >/opt/kubernetes/ssl/kube-proxy-csr.json << EOF
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

cd /opt/kubernetes/ssl/
cfssl gencert -ca=/opt/caCenter/ca.pem -ca-key=/opt/caCenter/ca-key.pem -config=/opt/caCenter/ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
```

依据CA证书生成管理员证书：
```
cat >/opt/kubernetes/ssl/admin-csr.json << EOF
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF

cd /opt/kubernetes/ssl/
cfssl gencert -ca=/opt/caCenter/ca.pem -ca-key=/opt/caCenter/ca-key.pem -config=/opt/caCenter/ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
```

依据CA证书生成kube-controller-manager证书
```
cat > /opt/kubernetes/ssl/kube-controller-manager-csr.json << EOF
{
    "CN": "system:kube-controller-manager",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "hosts": [
      "127.0.0.1",
      "192.168.0.15",
      "192.168.0.6",
      "10.255.0.1",
      "192.168.0.8"
    ],
    "names": [
      {
        "C": "CN",
        "ST": "Beijing",
        "L": "Beijing",
        "O": "system:kube-controller-manager",
        "OU": "system"
      }
    ]
}
EOF

cd /opt/kubernetes/ssl
cfssl gencert -ca=/opt/caCenter/ca.pem -ca-key=/opt/caCenter/ca-key.pem -config=/opt/caCenter/ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```

依据CA证书生成kube-scheduler-csr.json证书
```
cat > /opt/kubernetes/ssl/kube-scheduler-csr.json << EOF
{
    "CN": "system:kube-scheduler",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "hosts": [
      "127.0.0.1",
      "192.168.0.15",
      "192.168.0.6",
      "10.255.0.1",
      "192.168.0.8"
    ],
    "names": [
      {
        "C": "CN",
        "ST": "Beijing",
        "L": "Beijing",
        "O": "system:kube-scheduler",
        "OU": "system"
      }
    ]
}
EOF

cd /opt/kubernetes/ssl
cfssl gencert -ca=/opt/caCenter/ca.pem -ca-key=/opt/caCenter/ca-key.pem -config=/opt/caCenter/ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```

#### 组件部署准备
下载kubernetes二进制包(k8s-master1节点操作)
```
mkdir /opt/tempData -p
wget -P /opt/tempData https://dl.k8s.io/v1.23.4/kubernetes-server-linux-amd64.tar.gz
tar -zxf /opt/tempData/kubernetes-server-linux-amd64.tar.gz -C /opt/tempData
cp /opt/tempData/kubernetes/server/bin/{kube-apiserver,kube-scheduler,kube-controller-manager,kubectl} /usr/local/bin/
```

生成tocken文件(k8s-master1节点操作)
```
BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')
cat > /opt/kubernetes/cfg/token.csv <<EOF
${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF


cat /opt/kubernetes/cfg/token.csv

# 第一列：随机字符串，自己可生成
# 第二列：用户名
# 第三列：UID
# 第四列：用户组
```

#### 部署apiserver组件(k8s-master1节点操作)
创建apiserver配置文件
```
cat > /opt/kubernetes/cfg/kube-apiserver << EOF
KUBE_APISERVER_OPTS=" \
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
  --anonymous-auth=false \
  --bind-address=192.168.0.6 \
  --secure-port=6443 \
  --advertise-address=192.168.0.6 \
  --insecure-port=0 \
  --authorization-mode=Node,RBAC \
  --runtime-config=api/all=true \
  --enable-bootstrap-token-auth \
  --service-cluster-ip-range=10.255.0.0/24 \
  --token-auth-file=/opt/kubernetes/cfg/token.csv \
  --service-node-port-range=30000-50000 \
  --tls-cert-file=/opt/kubernetes/ssl/kube-apiserver.pem  \
  --tls-private-key-file=/opt/kubernetes/ssl/kube-apiserver-key.pem \
  --client-ca-file=/opt/caCenter/ca.pem  \
  --kubelet-client-certificate=/opt/kubernetes/ssl/kube-apiserver.pem \
  --kubelet-client-key=/opt/kubernetes/ssl/kube-apiserver-key.pem  \
  --service-account-key-file=/opt/caCenter/ca.pem \
  --service-account-signing-key-file=/opt/caCenter/ca-key.pem \
  --service-account-issuer=https://kubernetes.default.svc.cluster.local \
  --etcd-cafile=/opt/caCenter/ca.pem \
  --etcd-certfile=/opt/etcd/ssl/etcd.pem \
  --etcd-keyfile=/opt/etcd/ssl/etcd-key.pem \
  --etcd-servers=https://192.168.0.6:2379,https://192.168.0.8:2379,https://192.168.0.15:2379 \
  --enable-swagger-ui=true \
  --allow-privileged=true \
  --apiserver-count=3 \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/var/log/kube-apiserver-audit.log \
  --event-ttl=1h \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes \
  --v=2"
EOF

# --logtostderr：启用日志
# --v：日志等级
# --log-dir：日志目录
# --etcd-servers：etcd集群地址
# --bind-address：监听地址
# --secure-port：https安全端口
# --advertise-address：集群通告地址
# --allow-privileged：启用授权
# --service-cluster-ip-range：Service虚拟IP地址段
# --enable-admission-plugins：准入控制模块
# --authorization-mode：认证授权，启用RBAC授权和节点自管理
# --enable-bootstrap-token-auth：启用TLS bootstrap机制
# --token-auth-file：bootstrap token文件
# --service-node-port-range：Service nodeport类型默认分配端口范围
# --kubelet-client-xxx：apiserver访问kubelet客户端证书
# --tls-xxx-file：apiserver https证书
# --etcd-xxxfile：连接Etcd集群证书 –
# --audit-log-xxx：审计日志
```

systemd管理apiserver:
```
cat > /usr/lib/systemd/system/kube-apiserver.service << EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=etcd.service
Wants=etcd.service

[Service]
EnvironmentFile=-/opt/kubernetes/cfg/kube-apiserver
ExecStart=/usr/local/bin/kube-apiserver \$KUBE_APISERVER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

启动apiserver：
```
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl restart kube-apiserver
```

#### 部署kubectl工具(k8s-master1节点操作)
Kubectl是客户端工具，操作k8s资源的，如增删改查等
Kubectl操作资源的时候，默认先找变量`KUBECONFIG`，没有则就会使用`~/.kube/config`
可以设置一个环境变量`KUBECONFIG`：
```
export KUBECONFIG =/etc/kubernetes/admin.conf
```
这样在操作kubectl，就会自动加载`KUBECONFIG`来操作要管理哪个集群的k8s资源
也可以按照下面方法：
```
unset KUBECONFIG
cp /etc/kubernetes/admin.conf ~/.kube/config
```
这样我们在执行kubectl，就会加载`~/.kube/config`文件操作k8s资源

创建kubeconfig配置文件：
```
cd /opt/kubernetes/cfg
kubectl config set-cluster kubernetes --certificate-authority=/opt/caCenter/ca.pem --embed-certs=true --server=https://192.168.0.6:6443 --kubeconfig=kube.config
```

创建kubectl使用的用户admin，并指定了刚创建的证书和config：
```
kubectl config set-credentials admin --client-certificate=/opt/kubernetes/ssl/admin.pem --client-key=/opt/kubernetes/ssl/admin-key.pem --embed-certs=true --kubeconfig=kube.config
```

配置安全上下文：
```
kubectl config set-context kubernetes --cluster=kubernetes --user=admin --kubeconfig=kube.config
kubectl config use-context kubernetes --kubeconfig=kube.config
mkdir ~/.kube -p
cp /opt/kubernetes/cfg/kube.config ~/.kube/config
```

设置指令自动补全：
```
yum install -y bash-completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
exit # 重新登陆服务器
```

检查配置信息：
```
[root@k8s-master1 cfg]# kubectl cluster-info
Kubernetes control plane is running at https://192.168.0.6:6443

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
[root@k8s-master1 cfg]# kubectl get ns
NAME              STATUS   AGE
default           Active   22h
kube-node-lease   Active   22h
kube-public       Active   22h
kube-system       Active   22h

[root@k8s-master1 cfg]# kubectl get componentstatuses
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS      MESSAGE                                                                                        ERROR
scheduler            Unhealthy   Get "https://127.0.0.1:10259/healthz": dial tcp 127.0.0.1:10259: connect: connection refused   
controller-manager   Unhealthy   Get "https://127.0.0.1:10257/healthz": dial tcp 127.0.0.1:10257: connect: connection refused   
etcd-2               Healthy     {"health":"true","reason":""}                                                                  
etcd-1               Healthy     {"health":"true","reason":""}                                                                  
etcd-0               Healthy     {"health":"true","reason":""}  
```


#### 部署kube-controller-manager组件(k8s-master1节点操作)
创建kube-controller-manager的kubeconfig:
```
cd /opt/kubernetes/cfg
kubectl config set-cluster kubernetes --certificate-authority=/opt/caCenter/ca.pem --embed-certs=true --server=https://192.168.0.6:6443 --kubeconfig=kube-controller-manager.kubeconfig
```

创建用户：
```
cd /opt/kubernetes/cfg
kubectl config set-credentials system:kube-controller-manager --client-certificate=/opt/kubernetes/ssl/kube-controller-manager.pem --client-key=/opt/kubernetes/ssl/kube-controller-manager-key.pem --embed-certs=true --kubeconfig=kube-controller-manager.kubeconfig
```

添加用户安全上下文
```
cd /opt/kubernetes/cfg
kubectl config set-context system:kube-controller-manager --cluster=kubernetes --user=system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig  
```

切换用户安全上下文
```
cd /opt/kubernetes/cfg
kubectl config use-context system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig  
```


创建kube-controller-manager配置文件：
```
cat > /opt/kubernetes/cfg/kube-controller-manager << EOF
KUBE_CONTROLLER_MANAGER_OPTS=" \
  --secure-port=10257 \
  --bind-address=127.0.0.1 \
  --kubeconfig=/opt/kubernetes/cfg/kube-controller-manager.kubeconfig \
  --service-cluster-ip-range=10.255.0.0/24 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/opt/caCenter/ca.pem \
  --cluster-signing-key-file=/opt/caCenter/ca-key.pem \
  --allocate-node-cidrs=true \
  --cluster-cidr=10.0.0.0/16 \
  --experimental-cluster-signing-duration=876000h \
  --root-ca-file=/opt/caCenter/ca.pem \
  --service-account-private-key-file=/opt/caCenter/ca-key.pem \
  --leader-elect=true \
  --feature-gates=RotateKubeletServerCertificate=true \
  --controllers=*,bootstrapsigner,tokencleaner \
  --horizontal-pod-autoscaler-sync-period=10s \
  --tls-cert-file=/opt/kubernetes/ssl/kube-controller-manager.pem \
  --tls-private-key-file=/opt/kubernetes/ssl/kube-controller-manager-key.pem \
  --use-service-account-credentials=true \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes \
  --v=2"
EOF
```

systemd管理kube-controller-manager：
```
cat > /usr/lib/systemd/system/kube-controller-manager.service << EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=-/opt/kubernetes/cfg/kube-controller-manager
ExecStart=/usr/local/bin/kube-controller-manager \$KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

启动kube-controller-manager:
```
systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl restart kube-controller-manager
```

#### 部署kube-scheduler组件(k8s-master1节点操作)
创建kube-scheduler的kubeconfig：
```
cd /opt/kubernetes/cfg
kubectl config set-cluster kubernetes --certificate-authority=/opt/caCenter/ca.pem --embed-certs=true --server=https://192.168.0.6:6443 --kubeconfig=kube-scheduler.kubeconfig
```

创建用户：
```
cd /opt/kubernetes/cfg
kubectl config set-credentials system:kube-scheduler --client-certificate=/opt/kubernetes/ssl/kube-scheduler.pem --client-key=/opt/kubernetes/ssl/kube-scheduler-key.pem --embed-certs=true --kubeconfig=kube-scheduler.kubeconfig
```

添加用户安全上下文
```
cd /opt/kubernetes/cfg
kubectl config set-context system:kube-scheduler --cluster=kubernetes --user=system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig
```

切换用户安全上下文
```
cd /opt/kubernetes/cfg
kubectl config use-context system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig
```


创建kube-scheduler配置文件：
```
cat > /opt/kubernetes/cfg/kube-scheduler << EOF
KUBE_SCHEDULER_OPTS="--address=127.0.0.1 \
--kubeconfig=/opt/kubernetes/cfg/kube-scheduler.kubeconfig \
--leader-elect=true \
--alsologtostderr=true \
--logtostderr=false \
--log-dir=/var/log/kubernetes \
--v=2"
EOF
```

systemd管理kube-schduler：
```
cat > /usr/lib/systemd/system/kube-scheduler.service << EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=-/opt/kubernetes/cfg/kube-scheduler
ExecStart=/usr/local/bin/kube-scheduler \$KUBE_SCHEDULER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

启动kube-schduler：
```
systemctl daemon-reload
systemctl enable kube-scheduler
systemctl restart kube-scheduler
```

#### 其他master节点部署并启动kube-apiserver、kube-controller-manager、kube-scheduler
复制所需文件至k8s-master2、k8s-master3(k8s-master1节点操作)：
```
# 配置、证书
scp -r /opt/kubernetes root@k8s-master2:/opt/
scp -r /opt/kubernetes root@k8s-master3:/opt/

# systemd管理脚本
scp /usr/lib/systemd/system/{kube-controller-manager,kube-scheduler,kube-apiserver}.service root@k8s-master2:/usr/lib/systemd/system/
scp /usr/lib/systemd/system/{kube-controller-manager,kube-scheduler,kube-apiserver}.service root@k8s-master3:/usr/lib/systemd/system/

# 二进制文件
scp /usr/local/bin/{kube-apiserver,kube-controller-manager,kubectl,kube-scheduler}  root@k8s-master2:/usr/local/bin/
scp /usr/local/bin/{kube-apiserver,kube-controller-manager,kubectl,kube-scheduler}  root@k8s-master3:/usr/local/bin/

# KUBECONFIG
scp -r /root/.kube root@k8s-master2:/root
scp -r /root/.kube root@k8s-master3:/root
```

修改kube-apiserver的配置文件(k8s-master2、k8s-master3节点操作)：
```
currentIP=`hostname -I|awk '{print $1}'`
sed -i s'/192.168.0.6/'$currentIP'/g' /opt/kubernetes/cfg/kube-apiserver
```
启动k8s-master2、k8s-master3的kube-apiserver、kube-controller-manager、kube-scheduler服务(k8s-master2、k8s-master3节点操作)：
```
systemctl daemon-reload
systemctl enable kube-apiserver kube-controller-manager kube-scheduler
systemctl restart kube-apiserver kube-controller-manager kube-scheduler
```

## Node节点部署组件
#### 部署kubelet
创建kubelet-bootstrap.kubeconfig(k8s-master1操作)：
```
cd /opt/kubernetes/cfg
kubectl config set-cluster kubernetes --certificate-authority=/opt/caCenter/ca.pem --embed-certs=true --server=https://192.168.0.6:6443 --kubeconfig=kubelet-bootstrap.kubeconfig
```

创建用户(k8s-master1操作)：
```
BOOTSTRAP_TOKEN=$(awk -F "," '{print $1}' /opt/kubernetes/cfg/token.csv)
cd /opt/kubernetes/cfg
kubectl config set-credentials kubelet-bootstrap --token=${BOOTSTRAP_TOKEN} --kubeconfig=kubelet-bootstrap.kubeconfig
```

添加用户安全上下文(k8s-master1操作)：
```
cd /opt/kubernetes/cfg
kubectl config set-context default --cluster=kubernetes --user=kubelet-bootstrap --kubeconfig=kubelet-bootstrap.kubeconfig
```

切换用户安全上下文(k8s-master1操作)：
```
cd /opt/kubernetes/cfg
kubectl config use-context default --kubeconfig=kubelet-bootstrap.kubeconfig
```

用户权限设置(k8s-master1操作)：
```
cd /opt/kubernetes/cfg
kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
```

创建配置文件`kubelet`：(所有node节点操作)：
```
mkdir /opt/tempData
mkdir /opt/kubernetes/{cfg,ssl} -p
mkdir /opt/caCenter

scp k8s-master1:/opt/kubernetes/cfg/kubelet-bootstrap.kubeconfig /opt/kubernetes/cfg/
scp -r k8s-master1:/opt/caCenter/ca.pem /opt/caCenter
scp -r k8s-master1:/opt/tempData/kubernetes/server/bin/kubelet /usr/local/bin/

currentIP=`hostname -I|awk '{print $1}'`
cat > /opt/kubernetes/cfg/kubelet << EOF
{
  "kind": "KubeletConfiguration",
  "apiVersion": "kubelet.config.k8s.io/v1beta1",
  "authentication": {
    "x509": {
      "clientCAFile": "/opt/caCenter/ca.pem"
    },
    "webhook": {
      "enabled": true,
      "cacheTTL": "2m0s"
    },
    "anonymous": {
      "enabled": false
    }
  },
  "authorization": {
    "mode": "Webhook",
    "webhook": {
      "cacheAuthorizedTTL": "5m0s",
      "cacheUnauthorizedTTL": "30s"
    }
  },
  "address": "$currentIP",
  "port": 10250,
  "readOnlyPort": 10255,
  "cgroupDriver": "systemd",
  "hairpinMode": "promiscuous-bridge",
  "serializeImagePulls": false,
  "clusterDomain": "cluster.local.",
  "clusterDNS": ["10.255.0.2"]
}
EOF
```

systemd管理kubelet(所有node节点操作)：
```
dockerDisk=`df -Tk|grep -Ev "devtmpfs|tmpfs|overlay"|grep -E "ext4|ext3|xfs"|awk '/\//{print $5,$NF}'|sort -nr|awk '{print $2}'|head -1|tr '\n' ' '|awk '{print $1}'`
kubeletDataDir=`echo ${dockerDisk}/kubeletData|sed s'/\/\//\//g'`
mkdir ${kubeletDataDir}

cat > /usr/lib/systemd/system/kubelet.service << EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service
[Service]
WorkingDirectory=${kubeletDataDir}
ExecStart=/usr/local/bin/kubelet \
  --bootstrap-kubeconfig=/opt/kubernetes/cfg/kubelet-bootstrap.kubeconfig \
  --cert-dir=/opt/kubernetes/ssl \
  --root-dir=${kubeletDataDir}
  --kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \
  --config=/opt/kubernetes/cfg/kubelet \
  --network-plugin=cni \
  --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.6 \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes \
  --root-dir=${kubeletDataDir} \
  --v=2
Restart=on-failure
RestartSec=5
 
[Install]
WantedBy=multi-user.target
EOF
```

启动kubelet(所有node节点操作)：
```
systemctl daemon-reload
systemctl enable kubelet
systemctl restart kubelet
```

批准节点csr请求(k8s-master1节点操作)：
node节点启动后会向master节点发送csr请求，批准前的状态是`Pending`，批准之后的状态是`Approved,Issued`：
```
# 获取csr清单
csrName=`kubectl get csr|awk '/node/{print $1}'`

# 批准
kubectl certificate approve $csrName

# 查看csr状态
kubectl get csr
```

查看节点状态(k8s-master1节点操作)：
```
[root@k8s-master1 cfg]# kubectl get node
NAME        STATUS     ROLES    AGE     VERSION
k8s-node1   NotReady   <none>   19m     v1.23.4
k8s-node2   NotReady   <none>   2m32s   v1.23.4

# 由于还未部署网络插件，因此节点为NotReady
```


#### 部署kube-proxy
创建kube-proxy.kubeconfig(k8s-master1操作)：
```
cd /opt/kubernetes/cfg
kubectl config set-cluster kubernetes --certificate-authority=/opt/caCenter/ca.pem --embed-certs=true --server=https://192.168.0.6:6443 --kubeconfig=kube-proxy.kubeconfig
```

创建用户(k8s-master1操作)：
```
cd /opt/kubernetes/cfg
kubectl config set-credentials kube-proxy --client-certificate=/opt/kubernetes/ssl/kube-proxy.pem --client-key=/opt/kubernetes/ssl/kube-proxy-key.pem --embed-certs=true --kubeconfig=kube-proxy.kubeconfig

```

添加用户安全上下文(k8s-master1操作)：
```
cd /opt/kubernetes/cfg
kubectl config set-context default --cluster=kubernetes --user=kube-proxy --kubeconfig=kube-proxy.kubeconfig
```

切换用户安全上下文(k8s-master1操作)：
```
cd /opt/kubernetes/cfg
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

创建kube-proxy配置文件(所有node节点操作)：
```
scp k8s-master1:/opt/kubernetes/cfg/kube-proxy.kubeconfig /opt/kubernetes/cfg/
scp -r k8s-master1:/opt/tempData/kubernetes/server/bin/kube-proxy /usr/local/bin/

currentIP=`hostname -I|awk '{print $1}'`
# clusterCIDR 修改为集群的子网信息
cat > /opt/kubernetes/cfg/kube-proxy << EOF
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: ${currentIP}
clientConnection:
  kubeconfig: /opt/kubernetes/cfg/kube-proxy.kubeconfig
clusterCIDR: 10.0.0.0/16
healthzBindAddress: ${currentIP}:10256
kind: KubeProxyConfiguration
metricsBindAddress: ${currentIP}:10249
mode: "ipvs"
EOF
```

systemd管理kube-proxy(所有node节点操作)：
```
dockerDisk=`df -Tk|grep -Ev "devtmpfs|tmpfs|overlay"|grep -E "ext4|ext3|xfs"|awk '/\//{print $5,$NF}'|sort -nr|awk '{print $2}'|head -1|tr '\n' ' '|awk '{print $1}'`
kubeproxyDataDir=`echo ${dockerDisk}/kubeproxyData|sed s'/\/\//\//g'`
mkdir ${kubeproxyDataDir}
cat > /usr/lib/systemd/system/kube-proxy.service << EOF
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target
 
[Service]
WorkingDirectory=${kubeproxyDataDir}
ExecStart=/usr/local/bin/kube-proxy \
  --config=/opt/kubernetes/cfg/kube-proxy \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes \
  --v=2
Restart=on-failure
RestartSec=5
LimitNOFILE=65536
 
[Install]
WantedBy=multi-user.target
EOF
```

启动kube-proxy(所有node节点操作)：
```
systemctl daemon-reload
systemctl enable kube-proxy
systemctl restart kube-proxy
```

## 集群插件部署
#### 部署calico网络插件(k8s-master1节点操作)
下载calico编排文件：
```
wget -P /opt/tempData/ https://docs.projectcalico.org/manifests/calico.yaml
```

修改`calico.yaml`，确保`CALICO_IPV4POOL_CIDR`与`kube-controller-manager`配置文件中的`--cluster-cidr`保持一致
```
- name: CALICO_IPV4POOL_CIDR
  value: "10.0.0.0/16"
```

发布calico插件到集群：
```
kubectl apply -f /opt/tempData/calico.yaml
```

#### 部署DNS插件(k8s-master1节点操作)
复制链接文件中的内容到`/opt/tempData/coredns.yaml`：[链接](https://github.com/kubernetes/kubernetes/blob/master/cluster/addons/dns/coredns/coredns.yaml.sed)

修改配置：
```
sed -i s/\$DNS_SERVER_IP/10.255.0.2/g /opt/tempData/coredns.yaml
sed -i s/\$DNS_DOMAIN/cluster.local/g /opt/tempData/coredns.yaml
sed -i s/\$DNS_MEMORY_LIMIT/170Mi/g /opt/tempData/coredns.yaml
sed -i s#k8s.gcr.io/coredns/coredns:v1.8.6#registry.aliyuncs.com/google_containers/coredns:v1.8.6#g /opt/tempData/coredns.yaml
```
- `clusterIP: $DNS_SERVER_IP`，和`/opt/kubernetes/cfg/kubelet`中的`"clusterDNS": ["10.255.0.2"]`对应
- `$DNS_DOMAIN`和`apiserver`里的`host`中的域名对应`cluster.local`
- `memory: $DNS_MEMORY_LIMIT`大于下面的70Mi即可：
- image:k8s.gcr.io/coredns/coredns:v1.8.6 改成国内源地址


发布calico插件到集群：
```
kubectl apply -f /opt/tempData/coredns.yaml
```

## 配置高可用
#### 部署nginx负载均衡（所有节点操作）
下载并解压nginx：
```
wget -P /opt/tempData/ https://nginx.org/download/nginx-1.21.6.tar.gz
tar -zxf /opt/tempData/nginx-1.21.6.tar.gz -C /opt/tempData/
```

编译：
```
cd /opt/tempData/nginx-1.21.6 && ./configure --prefix=/opt/nginx --with-stream && make && make install

ln -s /opt/nginx/sbin/nginx /usr/local/bin/
```

配置：
```
rm -rf /opt/nginx/conf/*.default

cat > /opt/nginx/conf/nginx.conf << EOF
worker_processes  auto;
error_log  /var/log/nginx-error.log warn;

pid /run/nginx.pid;

events {
    worker_connections  1024;
}
        
stream {
    log_format main "\$remote_addr  \$upstream_addr  \$time_local \$status";
    upstream kube_apiserver {
        least_conn;
        server 192.168.0.15:6443 max_fails=3 fail_timeout=30s;
        server 192.168.0.6:6443 max_fails=3 fail_timeout=30s;
        server 192.168.0.8:6443 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 16443;
        access_log /var/log/nginx-access.log main;
        proxy_pass    kube_apiserver;
        proxy_timeout 10m;
        proxy_connect_timeout 10s;
    }
}
EOF
```

systemd管理nginx：
```
cat > /usr/lib/systemd/system/nginx.service << EOF
[Unit]
Description=The nginx HTTP and reverse proxy server
After=network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/run/nginx.pid
ExecStartPre=/usr/bin/rm -f /run/nginx.pid
ExecStartPre=/opt/nginx/sbin/nginx -t
ExecStart=/opt/nginx/sbin/nginx
ExecReload=/opt/nginx/sbin/nginx -s reload
KillSignal=SIGQUIT
TimeoutStopSec=5
KillMode=process
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF
```

启动nginx：
```
systemctl daemon-reload
systemctl enable nginx
systemctl restart nginx
```

#### 修改所有组件指向kube-apiserver的地址
所有master节点操作：
```
rm -rf /opt/kubernetes/cfg/kube.config

sed -i s'/192.168.0.6:6443/127.0.0.1:16443/g' ~/.kube/config
sed -i s'/192.168.0.6:6443/127.0.0.1:16443/g' /opt/kubernetes/cfg/kubelet
sed -i s'/192.168.0.6:6443/127.0.0.1:16443/g' /opt/kubernetes/cfg/kube-controller-manager.kubeconfig
sed -i s'/192.168.0.6:6443/127.0.0.1:16443/g' /opt/kubernetes/cfg/kubelet-bootstrap.kubeconfig
sed -i s'/192.168.0.6:6443/127.0.0.1:16443/g' /opt/kubernetes/cfg/kube-proxy.kubeconfig
sed -i s'/192.168.0.6:6443/127.0.0.1:16443/g' /opt/kubernetes/cfg/kube-scheduler.kubeconfig

systemctl restart kube-controller-manager kube-scheduler kube-apiserver
```

所有node节点操作：
```
sed -i s'/192.168.0.6:6443/127.0.0.1:16443/g' /opt/kubernetes/cfg/kubelet
sed -i s'/192.168.0.6:6443/127.0.0.1:16443/g' /opt/kubernetes/cfg/kubelet-bootstrap.kubeconfig
sed -i s'/192.168.0.6:6443/127.0.0.1:16443/g' /opt/kubernetes/cfg/kubelet.kubeconfig
sed -i s'/192.168.0.6:6443/127.0.0.1:16443/g' /opt/kubernetes/cfg/kube-proxy.kubeconfig
sed -i s'/192.168.0.6:6443/127.0.0.1:16443/g' /opt/kubernetes/cfg/kube-proxy

systemctl restart kube-proxy kubelet
```
 
## 测试
```
kubectl get nodes #确保所有节点ready状态

kubectl get pods -A # 确保所有pod running

tail -f /var/log/messages # 确保没有连接超时和拒绝的日志
```
