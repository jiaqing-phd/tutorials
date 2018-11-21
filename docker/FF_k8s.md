# k8s部署

如果不了解 k8s ，建议阅读 [三小时学会Kubernetes：容器编排详细指南](http://dockone.io/article/5132)

## 资源转换说明

- k8s 不支持 envfile 直接使用 .env 文件，需要对应的 env 文件转换为 configMap 文件
- docker-compose 中的 Service 名字可以以下划线连接，但是 k8s 的 DNS 不支持，需要转换为中划线 `-`
- 由于是集群化部署，k8s 虽然支持 hostPath 挂载方式，但是不推荐使用，所以需要将简单挂载的单个文件转为 configMap 或者 secret 资源，然后以文件方式挂载到 Pod 之中，如时区文件转为 configMap 文件 timezone-config.yaml，initdb.sql 转换为 mysql-config.yaml 文件中的 INITDB 项
- k8s 中要想引入非初始化到时候指定的镜像源，需要使用特殊的 secret 文件保存私有仓库的用户登录信息，这里的是 docker-secret.yaml，使用的是 docker.hashworld.top 的 prod 的一个只读账号 aliyun_k8s
- 其他需要挂载数据卷的地方过多，这里做出了简化，日志文件使用的卷为 Node 上的临时卷，所以之后的工作还需做到将 Pod 的前台日志和 Pod 里的 logging 日志通过中心日志服务收集存储，保证临时卷的销毁不会导致日志的丢失
- 其他需要挂载的数据卷如 local MySQL的数据，就需要使用 PVC/PV 结合阿里 NAS 盘通过 NFS 挂载

## 本地 k8s 环境搭建

为了方便在本机搭建多机虚拟机的实验环境，需要安装 `minikube` 这个就类似于 `docker-machine`，其实是复用了 `docker-machine`，所以支持的虚拟机也一样，只不过 `minikube` 采用的是 `Minikube.iso` 镜像，`docker-machine` 采用的是 `boot2docker.iso`

安装 minikube 并启动，默认是 virtualbox 来启动的，所以需要提前安装 virtualbox，可以使用 `--vm-driver` 参数指定

```shell
$ brew cask install minikube
$ minikube start --vm-driver=virtualbox  --registry-mirror=https://registry.docker-cn.com
```

如果上述的操作卡在了 `starting cluster components` 时候，需要参考[安装minkube 0.25](https://www.cnblogs.com/lynsyklate/p/9440519.html) 重新安装

minikube 安装时候同时也安装了 kubernetes-cli 这时候就可以使用 kubectl 命令，kubectl 是管理 k8s 的管理器

kubectl 和 docker 一样也是，基于 server 和 client 的结构，通过 `kubectl version` 可以看到，所以 kubectl 除了可以连上上 minikube 的 APIServer 还可以连接到阿里云的创建的 k8s

### 运行一下

使用 `minikube dashboard` 命令开启 dashboard 即可看见本地的 k8s 管理界面

## 创建集群

如果使用阿里云的 k8s 需要指向下面的步骤，如果只在本地使用 minikube 验证可以跳过下面步骤

为了便于快速创建，这里使用 kubectl 的方式连接上阿里的 k8s 进行资源操作，调整好资源项(如果测试的话可以选用最低配的磁盘和vps)其他按照默认创建 k8s 集群。

创建之后，要想使用 kubectl 连接上阿里 k8s，备份一份本地的 kube/config 配置然后参考 [连接k8s](https://help.aliyun.com/document_detail/86494.html) 将本地 kubectl 连接到阿里云 k8s 上。

## 构建镜像

拉取 `git clone https://github.com/McGrady00H/hashworld-cluster.git` 代码，切换到 k8s 分支(暂时)，参考 [cluster README](https://github.com/McGrady00H/hashworld-cluster) 构建镜像

默认 build 的 tag 格式是 `docker.hashworld.top/dev/hashfuture_asset_blockchain:$(TAG)`  其中 `$(TAG)` 为随机生成的 tag，在 k8s 中使用的标签为 `docker.hashworld.top/prod/XXX` 所以在本机测试完毕之后，需要将测试完毕的 `dev` 镜像使用 `docker tag` 语句标记镜像为 `docker.hashworld.top/prod/XXX`

之后使用 `docker login doccker.hashworld.top` 登录私有镜像库，使用 `docker push docker.hashworld.top/prod/XXX` 将镜像推送到私有镜像

如果使用本地的 minikube 可以将 docker 连接到 minikube 的 docker engine 直接在 minikube 虚拟机上构建镜像

## 启动

首先创建一些配置资源，如果需要修改配置，需要这一步之前完成

```shell
$ cd kubernetes
$ kubectl create -f docker-secret.yaml
$ kubectl cretae -f configMap
```

### 数据卷创建

如果在阿里云上使用，需要在阿里云上创建一个 NAS 云盘(这里的云盘的区最好和k8s集群里的一样)，[参考](https://help.aliyun.com/document_detail/88940.html)，需要手动创建三个 存储卷/存储声明(PV/PVC) 名称需要对应 mysql-data、hashfuture-asset-backend-local、hashfuture-account-micro-local，创建 PV 时候需要使用子目录进行挂载，不然一个 NAS 云盘可能会导致只能创建一对 PV/PVC

如果是本地的 minikube 创建，执行 `kubectl cretae -f minikube` 即可

### 创建 Pod Svc

```shell
$ kubectl cretae -f deployment
```

之后在阿里的管理界面/dashboard 就可以看见各个 **部署** 启动的情况

## 更新

一般镜像更新步骤

- 修改相应的工程的代码
- 重新构建镜像
- 为新镜像打上特定的 tag，push 该 tag 的镜像到私有仓库
- 修改 kubernetes/deployment 文件夹下的的该工程的 yaml 文件的镜像的 tag
- 使用 `kubectl apply -f xxxx.yaml` 更新资源
