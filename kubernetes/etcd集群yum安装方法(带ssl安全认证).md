### 安装etcd三节点集群(所有master节点执行)
```
yum -y install etcd
```
### 为保证内网etcd访问的安全性，为其配置安全证书
### 使用cfssl来生成自签证书(master1执行)
```
mkdir /etc/etcd/cert -v
curl -L https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -o /usr/local/bin/cfssl
curl -L https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -o /usr/local/bin/cfssljson
curl -L https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -o /usr/local/bin/cfssl-certinfo
chmod +x /usr/local/bin/cfssl /usr/local/bin/cfssljson /usr/local/bin/cfssl-certinfo

```
### 创建文件：`ca-config.json`,`ca-csr.json`,`server-csr.json`(master1执行)
```
cat > /etc/etcd/cert/ca-config.json  << EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "www": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
  "expiry": "87600h"
      }
    }
  }
}
EOF

cat > /etc/etcd/cert/ca-csr.json  << EOF
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "ShenZhen",
            "ST": "ShenZhen",
      "O": "k8s",
            "OU": "System"
        }
    ],
    "ca": {
  "expiry": "87600h"
    }
}
EOF

cat > /etc/etcd/cert/server-csr.json  << EOF
{
    "CN": "etcd",
    "hosts": [
    "127.0.0.1",
    "10.8.20.79",
    "10.8.20.58",
    "10.8.20.59",
    "10.254.0.1",
    "master1",
    "master2",
    "master3",
    "apiserver1",
    "apiserver2",
    "apiserver3",
    "etcd1",
    "etcd2",
    "etcd3",
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
            "L": "ShenZhen",
            "ST": "ShenZhen",
      "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF

  #可以看到该证书把etcd集群的所有ip,kubernetes master的所有ip以及kubernetes服务的ip(10.254.0.1)都加入进去了，这样他们都能使用同一个密钥

```
### 生成证书(master1执行)
```
cd /etc/etcd/cert
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server
```
### 将生成的证书复制到其他etcd集群节点(master1执行)
```
scp -r /etc/etcd/cert etcd1:/etc/etcd
scp -r /etc/etcd/cert etcd2:/etc/etcd
scp -r /etc/etcd/cert etcd3:/etc/etcd
```
### 配置etcd服务启动脚本(所有master节点执行)
```
cat > /usr/lib/systemd/system/etcd.service << EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
User=etcd
# set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=\$(nproc) \
/usr/bin/etcd --name=\"\${ETCD_NAME}\" \
--data-dir=\"\${ETCD_DATA_DIR}\" \
--listen-peer-urls=\"\${ETCD_LISTEN_PEER_URLS}\" \
--listen-client-urls=\"\${ETCD_LISTEN_CLIENT_URLS}\" \
--advertise-client-urls=\"\${ETCD_ADVERTISE_CLIENT_URLS}\" \
--initial-cluster-token=\"\${ETCD_INITIAL_CLUSTER_TOKEN}\" \
--initial-cluster=\"\${ETCD_INITIAL_CLUSTER}\" \
--initial-cluster-state=\"\${ETCD_INITIAL_CLUSTER_STATE}\" \
--cert-file=/etc/etcd/cert/server.pem \
--key-file=/etc/etcd/cert/server-key.pem \
--peer-cert-file=/etc/etcd/cert/server.pem \
--peer-key-file=/etc/etcd/cert/server-key.pem \
--trusted-ca-file=/etc/etcd/cert/ca.pem \
--peer-trusted-ca-file=/etc/etcd/cert/ca.pem"

Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```
### 配置etcd配置文件/etc/etcd/etcd.conf(master1上执行)
### 接口URL不支持域名，注意替换具体的IP
```
mv -v /etc/etcd/etcd.conf{,.bak}
cat > /etc/etcd/etcd.conf << EOF
ETCD_NAME=etcd1
ETCD_DATA_DIR="/var/lib/etcd/etcd1"
ETCD_LISTEN_PEER_URLS="https://10.8.20.79:2380"
ETCD_LISTEN_CLIENT_URLS="https://127.0.0.1:2379,https://10.8.20.79:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.8.20.79:2380"
ETCD_INITIAL_CLUSTER="etcd1=https://10.8.20.79:2380,etcd2=https://10.8.20.58:2380,etcd3=https://10.8.20.59:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="LCjJgRjfN2fIARYb"
ETCD_ADVERTISE_CLIENT_URLS="https://10.8.20.79:2379"
EOF
```
#配置etcd配置文件/etc/etcd/etcd.conf(master2上执行)
```
mv -v /etc/etcd/etcd.conf{,.bak}
cat > /etc/etcd/etcd.conf << EOF
ETCD_NAME=etcd2
ETCD_DATA_DIR="/var/lib/etcd/etcd2"
ETCD_LISTEN_PEER_URLS="https://10.8.20.58:2380"
ETCD_LISTEN_CLIENT_URLS="https://127.0.0.1:2379,https://10.8.20.58:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.8.20.58:2380"
ETCD_INITIAL_CLUSTER="etcd1=https://10.8.20.79:2380,etcd2=https://10.8.20.58:2380,etcd3=https://10.8.20.59:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="LCjJgRjfN2fIARYb"
ETCD_ADVERTISE_CLIENT_URLS="https://10.8.20.58:2379"
EOF
```
### 配置etcd配置文件/etc/etcd/etcd.conf(master3上执行)
```
mv -v /etc/etcd/etcd.conf{,.bak}
cat > /etc/etcd/etcd.conf << EOF
ETCD_NAME=etcd3
ETCD_DATA_DIR="/var/lib/etcd/etcd3"
ETCD_LISTEN_PEER_URLS="https://10.8.20.59:2380"
ETCD_LISTEN_CLIENT_URLS="https://127.0.0.1:2379,https://10.8.20.59:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.8.20.59:2380"
ETCD_INITIAL_CLUSTER="etcd1=https://10.8.20.79:2380,etcd2=https://10.8.20.58:2380,etcd3=https://10.8.20.59:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="LCjJgRjfN2fIARYb"
ETCD_ADVERTISE_CLIENT_URLS="https://10.8.20.59:2379"
EOF
```
### 启动etcd服务(所有master节点依次执行)
```
chown etcd.etcd  -R /etc/etcd
systemctl daemon-reload
systemctl restart etcd
systemctl enable etcd
```
### 查看etcd集群的成员信息,不带证书会报错
```
[root@master1 cert]# etcdctl --ca-file=/etc/etcd/cert/ca.pem --cert-file=/etc/etcd/cert/server.pem --key-file=/etc/etcd/cert/server-key.pem --endpoints="https://etcd1:2379,https://etcd2:2379,https://etcd3:2379" member list
16d59e6d56a547a: name=etcd1 peerURLs=https://10.8.20.79:2380 clientURLs=https://10.8.20.79:2379 isLeader=true
8282d9a5229b73bb: name=etcd2 peerURLs=https://10.8.20.58:2380 clientURLs=https://10.8.20.58:2379 isLeader=false
839b97cccea6de9e: name=etcd3 peerURLs=https://10.8.20.59:2380 clientURLs=https://10.8.20.59:2379 isLeader=false


  #从列出信息可以看出，目前是etcd1为主节点。

  #查看etcd服务启动日志，可通过tail -f /var/log/messages动态查看
```
