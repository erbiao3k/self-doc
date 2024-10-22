#### helm介绍
Helm是一个Kubernetes的包管理工具，就像Linux下的包管理器，如yum/apt等，可以很方便的将之前打包好的yaml文件部署到kubernetes上。

Helm有3个重要概念：
- helm：一个命令行客户端工具，主要用于Kubernetes应用chart的创建、打包、发布和管理。
- Chart：应用描述，一系列用于描述k8资源相关文件的集合。
- Release：基于Chart的部署实体，一个chart被Helm运行后将会生成对应的一个 release；将在k8s中创建出真实运行的资源对象。

#### helm v3
- 不再使用tiller，转而使用kubeconfig连接kubernets集群
- Release名称可在不同命名空间重用
- 支持将Chart推送至Docker镜像仓库中
- ...

#### helm部署
```
cd /tmp
wget https://get.helm.sh/helm-v3.5.2-linux-amd64.tar.gz
tar -zxvf helm-v3.5.2-linux-amd64.tar.gz 
mv linux-amd64/helm /usr/bin/helm
chmod 700 /usr/bin/helm
```

#### 常用指令
**仓库管理**
```
# 添加helm仓库bitnami到本地
helm repo add bitnami	https://charts.bitnami.com/bitnami --username ${username} --password ${password}

#列出本地仓库
helm repo list

#删除本地仓库
helm repo remove bitnami
```

**搜索仓库**
```
# 在公网公共仓库搜索helm关键字
helm search hub helm

# 在本地仓库搜索helm
helm search repo helm
```

**下载仓库中的应用**
```
# 下载bitnami仓库中的应用common，版本为1.3.9
helm pull bitnami/common --version 1.3.9

#不指定版本默认latest
helm pull bitnami/common

```

**创建一个新应用**
```
helm create foo

    foo/
    ├── .helmignore   # Contains patterns to ignore when packaging Helm charts.
    ├── Chart.yaml    # Information about your chart
    ├── values.yaml   # The default values for your templates
    ├── charts/       # Charts that this chart depends on
    └── templates/    # The template files
        └── tests/    # The test files
```
**检查应用语法**
```
helm lint foo

1 chart(s) linted, 0 chart(s) failed
```

**渲染应用**
```
helm template foo
```

**模拟发布release**
```
helm install --dry-run --debug --namespace  ${namespace}  release-name  foo

#--dry-run 模拟安装
#--debug   打印调试信息
```

**发布release**
```
helm install --debug --namespace  ${namespace}  release-name  foo
```

**更新release**
```
 helm upgrade --debug  --install  --namespace ${namespace} -f values.yaml release-name foo

#名称为release-name的应用不存在会创建新的release-name应用

#--dry-run 也适用
```

**卸载release**
```
# 卸载且删除
helm uninstall  --purge  --debug  ${chartname}  --namespace ${namespace}
```

**上传应用到仓库(harbor、nexus等)**
```
# 先安装上传插件
helm plugin install https://github.com/chartmuseum/helm-push
helm push foo repo-name
```

**查询已发布release的发布历史**
```
helm history -n ${namespace} release-name

REVISION    UPDATED                     STATUS          CHART             APP VERSION     DESCRIPTION
1           Mon Oct 3 10:15:13 2016     superseded      alpine-0.1.0      1.0             Initial install
2           Mon Oct 3 10:15:13 2016     superseded      alpine-0.1.0      1.0             Upgraded successfully
3           Mon Oct 3 10:15:13 2016     superseded      alpine-0.1.0      1.0             Rolled back to 2
4           Mon Oct 3 10:15:13 2016     deployed        alpine-0.1.0      1.0             Upgraded successfully

```

**回滚release**
```
#查询release的历史版本 [REVISION]
helm history -n ${namespace} release-name

# 回滚
helm rollback -n ${namespace} release-name [REVISION] [flags]

Flags:
      --cleanup-on-fail    allow deletion of new resources created in this rollback when rollback fails
      --dry-run            simulate a rollback
      --force              force resource update through delete/recreate if needed
  -h, --help               help for rollback
      --history-max int    limit the maximum number of revisions saved per release. Use 0 for no limit (default 10)
      --no-hooks           prevent hooks from running during rollback
      --recreate-pods      performs pods restart for the resource if applicable
      --timeout duration   time to wait for any individual Kubernetes operation (like Jobs for hooks) (default 5m0s)
      --wait               if set, will wait until all Pods, PVCs, Services, and minimum number of Pods of a Deployment, StatefulSet, or ReplicaSet are in a ready state before marking the release as successful. It will wait for as long as --timeout
      --wait-for-jobs      if set and --wait enabled, will wait until all Jobs have been completed before marking the release as successful. It will wait for as long as --timeout
```

#### helm的一般玩法
**创建一个新应用**
```
helm create foo

    foo/
    ├── .helmignore   # Contains patterns to ignore when packaging Helm charts.
    ├── Chart.yaml    # Information about your chart
    ├── values.yaml   # The default values for your templates
    ├── charts/       # Charts that this chart depends on
    └── templates/    # The template files
        └── tests/    # The test files
```
按照上述的目录结构，一般在**values.yaml**中定义一个应用的关键信息，像：
- 镜像信息
- service端口信息
- ingress
- 存储
- ...

定义好后需要描述当前应用的版本信息，**一个版本就是一个release**，release的信息会在Chart.yaml中描述，必要的有：
- apiVersion: v2 # helm api版本，无需修改
- appVersion: v20200706-1044 # 当前应用的版本，我一般直接用镜像的tag
- description: # 应用描述
- name: common-service # 应用描述
- version: 1.0.0 # 当前release的版本。**在upgrade时此版本要比已经发布过的版本新才行**。

应用的具体编排文件会放在templates，比如deployment、service、configmap、pv、ingress等等资源的编排文件。templates下的编排文件正如目录所述，都是模板，是一个个资源的编排逻辑，不是具体的编排文件。在部署release时，会按照**values.yaml**中的关键信息，渲染出具体的编排文件，安装到集群。

**实际应用过程中，一般会生成一个通用的chart(网上有一些比较好的通用方案可以套用)，然后使用jenkins或者shell**（自动或手动）**生成一个具体的应用**。当然，在生成过程中，可能需要传入一些参数。如：
![](https://images.cnblogs.com/cnblogs_com/erbiao/1918053/o_210216083037helm-create-jenkins.png)
