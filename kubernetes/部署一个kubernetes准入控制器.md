## 部署说明
为了让使用者更加安全，规范的使用kubernetes，我们可以在kubernetes自定义一个准入控制器。总的来说，准入控制器主要可以：
- 安全性：准入控制器可以通过在整个命名空间或集群中，强制使用合理的安全基准来提高安全性。内置的PodSecurityPolicy准入控制器可能是最突出的例子；例如，它可以用于禁止容器以root身份运行，或者确保容器的根文件系统始终以只读方式挂载。可通过基于webhook的自定义准入控制器实现的其他用例包括：

  允许仅从企业已知的特定仓库中提取镜像，同时拒绝未知的镜像仓库。

  拒绝不符合安全标准的部署。例如，使用特权（privileged）标志的容器可以规避许多安全检查。基于webhook的准入控制器可以减轻此风险，该准入控制器拒绝此类部署（验证）或覆盖特权（privileged）标志，将其设置为false。

- 治理：准入控制器允许你强制遵守某些做法，例如具有良好的标签、注释、资源限制或其他设置。一些常见的场景包括：

  对不同对象强制执行标签验证，以确保将正确的标签用于各种对象，例如分配给团队或项目的每个对象，或指定应用程序标签的每个部署。

  自动向对象添加注释，例如为“dev”部署资源分配正确的成本中心。

- 配置管理：准入控制器允许你验证群集中运行对象的配置，并防止群集中任何明显的错误配置。准入控制器可用于检测和修复没有语义标签的部署镜像，例如：

  自动添加资源限制或验证资源限制，

  确保合理的标签被添加到pod，或

  确保生产部署中使用的镜像引用不使用最新的（latest）标记或带有-dev后缀的标记。

## 准备工作
#### 配置webhook接口证书
```
# 下载证书生成工具
wget -O /usr/local/bin/cfssl https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssl_1.6.4_linux_amd64
wget -O /usr/local/bin/cfssljson  https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssljson_1.6.4_linux_amd64

mkdir /data/admission-registry-ssl -p
cd /data/admission-registry-ssl

# 创建CA证书机构
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "876000h"
    },
    "profiles": {
      "server": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "876000h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
    "CN": "kubernetes",
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

# 生成 CA 证书和私钥
cfssl gencert -initca ca-csr.json | cfssljson -bare ca

# 此处我们将控制器放在default命名空间下，服务名称为admission-registry
ServiceName=admission-registry
ServiceNamespace=default

# 创建Server端证书，其中最重要的就是 -hostname 的值，格式为${ServiceName}.${ServiceNamespace}.svc，其中${ServiceName}代表你 webhook 的 Service 名字，${ServiceNamespace}代表你 webhook 的命名空间。
cat > server-csr.json <<EOF
{
  "CN": "admission",
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

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -hostname=${ServiceName}.${ServiceNamespace}.svc -profile=server server-csr.json | cfssljson -bare server

# 生成的 server 证书和私钥创建一个 Secret 对象，后面我们通过 Volumes 的形式将 Secret 挂载到 webhook 的容器中指定的位置给 webhook 使用
kubectl -n ${ServiceNamespace} create secret tls ${ServiceName}-tls --key=server-key.pem --cert=server.pem
```

#### docker镜像
```
mkdir /data/admission-registry-docker -p
cd /data/admission-registry-docker

cat >Dockerfile <<EOF
FROM golang:1.20 as builder

WORKDIR /admission-registry

ENV CGO_ENABLED=0
ENV GOOS=linux
ENV GOARCH=amd64
ENV GO111MODULE=on
ENV GOPROXY="https://goproxy.cn"

ADD ./admission-registry .

RUN go build -a -o admission-registry main.go

FROM alpine:3.9.2
COPY --from=builder /admission-registry/admission-registry .
ENTRYPOINT ["/admission-registry"]
EOF

yourDockerHub=hub.yourhub.com
imageTag=${yourDockerHub}/admission-registry:v1.0.0
docker build -t  ${imageTag} .
docker push  ${imageTag}
```

#### 部署webhook
##### 使用 Deployment + Service 来提供服务，在 Pod 的规范中配置环境变量 WHITELIST_REGISTRIES 来定义白名单镜像仓库地址，配置环境变量WHITELIST_REPOSITORIES来定义镜像仓库白名单，主要防止测试镜像发生产。然后将证书通过 Secret 的 Volumes 形式挂载到 Pod 容器中。
```
whitelistRegistries="hub.yourhub.com,hub.yourhub2.com"
whitelistRepositories="prod,v2-prod"

cat > app.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admission-registry
  labels:
    app: admission-registry
spec:
  selector:
    matchLabels:
      app: admission-registry
  template:
    metadata:
      labels:
        app: admission-registry
    spec:
      containers:
        - name: admission-registry
          image: ${yourDockerHub}:admission-registry:v1.0.0
          imagePullPolicy: IfNotPresent
          env:
            - name: WHITELIST_REGISTRIES
              value: ${whitelistRegistries}
            - name: WHITELIST_REPOSITORIES
              value: ${whitelistRepositories}
          ports:
          - containerPort: 443
          volumeMounts:
          - name: webhook-certs
            mountPath: /etc/webhook/certs
            readOnly: true
      volumes:
        - name: webhook-certs
          secret:
            secretName: admission-registry-tls
---
apiVersion: v1
kind: Service
metadata:
  name: admission-registry
  labels:
    app: admission-registry
spec:
  ports:
  - port: 443
    targetPort: 443
  selector:
    app: admission-registry
EOF

kubectl apply -f app.yaml -n ${ServiceNamespace}
```

#### 注册webhook
##### 上面我们只是单纯将我们实现的 webhook 部署到了 Kubernetes 集群中，但是还并没有和`ValidatingWebhook`对接起来，要将我们上面实现的服务注册到`ValidatingWebhook`中只需要创建一个类型为`ValidatingWebhookConfiguration`的Kubernetes 资源对象即可，在这个对象中就可以来配置我们的 webhook 这个服务。
如下所示，我们将 webhook 命名为`io.ydzs.admission-registry`，`只需要保证在集群中名称唯一`。然后在 rules 属性下面就是来指定在什么条件下使用该 webhook 的配置，这里我们只需要在创建 Pod 的时候才调用这个 webhook。此外在 ClientConfig 属性下我们还需要指定 Kubernetes APIServer 如何来找到我们的 webhook 服务，这里我们将通过一个在 default 命名空间下面的名为 admission-registry 的 Service 服务在 /validate 路径下面提供服务，此外还指定了一个 caBundle 的属性，这个属性通过指定一个 PEM 格式的 CA bundle 来表示 APIServer 作为客户端可以使用它来验证我们的 webhook 应用上的服务器证书。对应的注册 webhook 的资源清单如下所示：
```
# CA_BUNDLE值使用的是上面生成 ca.pem 文件内容的 base64 值
CA_BUNDLE=`cat /data/admission-registry-ssl/ca.pem | base64 -w 0`
webhookNameValidate=io.ydzs.admission-registry
webhookNameMutate=io.ydzs.admission-registry-mutate
cat > validatingwebhook.yaml << EOF
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: admission-registry
webhooks:
  - name: '${webhookNameValidate}'
    rules:
      - apiGroups:
          - ''
        apiVersions:
          - v1
        operations:
          - CREATE
          - DELETE
          - UPDATE
        resources:
          - pods
    clientConfig:
      service:
        namespace: '${ServiceNamespace}'
        name: admission-registry
        path: /validate
      caBundle: '${CA_BUNDLE}'
    admissionReviewVersions:
      - v1
    sideEffects: None
---
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: admission-registry-mute
webhooks:
  - name: '${webhookNameMutate}'
    clientConfig:
      service:
        namespace: '${ServiceNamespace}'
        name: admission-registry
        path: /mutate
      caBundle: '${CA_BUNDLE}'
    rules:
      - operations:
          - CREATE
          - DELETE
          - UPDATE
        apiGroups:
          - apps
          - ''
        apiVersions:
          - v1
        resources:
          - deployments
          - services
    admissionReviewVersions:
      - v1
    sideEffects: None
EOF

kubectl apply -f validatingwebhook.yaml -n ${ServiceNamespace}

kubectl get validatingwebhookconfiguration  -n ${ServiceNamespace}
# NAME                             WEBHOOKS   AGE
# admission-registry               1          52s
```

##### 参考文档
```
1、准入控制器参考：[链接](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/admission-controllers/)
2、Kubernetes准入控制器指南：[链接](https://mp.weixin.qq.com/s?__biz=MzI5ODk5ODI4Nw==&mid=2247486706&idx=2&sn=874ff77e78fed6e1030e3dd723c47bc4&chksm=ec9c0392dbeb8a84f3741621272decf1067e2c1b26833408c0a3fc6c0db3879b7ce73646a61b&mpshare=1&scene=1&srcid=0603Ysb7iweoNyupNAUyPrBk&sharer_sharetime=1685775624286&sharer_shareid=88f8081c63c394d21fd0a309a6e42be3&version=4.1.2.70182&platform=mac#rd)
3、实现一个容器镜像白名单的准入控制器 | 视频文字稿：[链接](https://mp.weixin.qq.com/s?src=11&timestamp=1685945554&ver=4571&signature=nyLRcKRc5fwaaQGh93y-WnJ1IAh3aXgqJyBGJNGz-Yy*7PmO5vin4X43EAmrHcjvrTjpUE1LMhGKPG0fmfOXc8vkMcxISnPf1E7sYcjyWhPqDHLCW-4pqeqt*adgwZhk&new=1)
```
