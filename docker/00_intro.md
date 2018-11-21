# docker笔记

## 误区

### sudo

非 root 用户在执行 docker 命令的时候总会提示权限不够，这时候一般要加上 sudo 才行，非常麻烦，其实可以通过将该用户添加到 docker 用户组来解决这个问题，如命令 `sudo usermod -aG docker $USER`

### volume

按照 Docker 最佳实践的要求，容器不应该向其存储层内写入任何数据，容器存储层要保持无状态化。所有的文件写入操作，都应该使用 数据卷（Volume）、或者绑定宿主目录，在这些位置的读写会跳过容器存储层，直接对宿主（或网络存储）发生读写，其性能和稳定性更高。

### commit

`docker commit` 命令有一些特殊的应用场合，比如被入侵后保存现场等。但是，不要使用 `docker commit` 定制镜像，定制镜像应该使用 Dockerfile 来完成，

要知道，当我们运行一个容器的时候（如果不使用卷的话），我们做的任何文件修改都会被记录于容器存储层里。而 Docker 提供了一个 `docker commit` 命令，可以将容器的存储层保存下来成为镜像。换句话说，就是在原有镜像的基础上，再叠加上容器的存储层，并构成新的镜像。以后我们运行这个新镜像的时候，就会拥有原有容器最后的文件变化。

此外，使用 docker commit 意味着所有对镜像的操作都是黑箱操作，生成的镜像也被称为黑箱镜像，换句话说，就是除了制作镜像的人知道执行过什么命令、怎么生成的镜像，别人根本无从得知。

### Go

如果你以 scratch 为基础镜像的话，意味着你不以任何镜像为基础，接下来所写的指令将作为镜像第一层开始存在。

不以任何系统为基础，直接将可执行文件复制进镜像的做法并不罕见，比如 swarm、coreos/etcd。对于 Linux 下静态编译的程序来说，并不需要有操作系统提供运行时支持，所需的一切库都已经在可执行文件里了，因此直接 FROM scratch 会让镜像体积更加小巧。使用 Go 语言 开发的应用很多会使用这种方式来制作镜像，这也是为什么有人认为 Go 是特别适合容器微服务架构的语言的原因之一

### daemon

Docker 不是虚拟机，容器中的应用都应该以前台执行，而不是像虚拟机、物理机里面那样，用 `upstart/systemd` 去启动后台服务，容器内没有后台服务的概念。

如果使用 `CMD service nginx start` 然后发现容器执行后就立即退出了，对于容器而言，其启动程序就是容器应用进程，容器就是为了主进程而存在的，主进程退出，容器就失去了存在的意义，从而退出，其它辅助进程不是它需要关心的东西。`CMD service nginx start` 会被理解为 `CMD [ "sh", "-c", "service nginx start"]`，因此主进程实际上是 `sh`。那么当 `service nginx start` 命令结束后，`sh` 也就结束了，`sh` 作为主进程退出了，自然就会令容器退出。

正确的做法是直接执行 nginx 可执行文件，并且要求以前台形式运行。比如：
```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

### Run

每一句 `RUN` 都会创建一层，所以争取的方式示例如下，清理工作的命令，删除了为了编译构建所需要的软件，清理了所有下载、展开的文件，并且还清理了 `apt` 缓存文件

```dockerfile
FROM debian:jessie

RUN buildDeps='gcc libc6-dev make' \
    && apt-get update \
    && apt-get install -y $buildDeps \
    && wget -O redis.tar.gz "http://download.redis.io/releases/redis-3.2.5.tar.gz" \
    && mkdir -p /usr/src/redis \
    && tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1 \
    && make -C /usr/src/redis \
    && make -C /usr/src/redis install \

    && rm -rf /var/lib/apt/lists/* \
    && rm redis.tar.gz \
    && rm -r /usr/src/redis \
    && apt-get purge -y --auto-remove $buildDeps
```

`RUN` 命令并不等于Shell脚本，又如下面的例子，最终在 `/app`目录下是找不到创建的 `world.txt` 文件的，因为这两个 `RUN` 命令执行环境不同，因为不属于同层的操作，所以上下文的pwd是不同步的，但是可以通过设置 `WORKDIR` 来指定

```dockerfile
RUN cd /app
RUN echo "hello" > world.txt
```

## 入门

修改 Docker registry，即docker的镜像源，参考私有仓库部分，修改完之后执行 `docker info` 来查看信息

使用 `docker pull ubuntu:16.04` 拉取一个镜像，之后使用该镜像创建一个运行实例-容器 `docker run -it --rm ubuntu:16.04 bash` 并进入该容器的bash终端，退出终端后，容器会被删除

## 镜像命令

- `docker image ls` 列出本地拉取的镜像，加上 `-a` 参数显示中间层镜像
- `docker image rm <image>` 其中\<image\>可以为 `部分ID`、`镜像名`
- `docker system df` 查看镜像、容器、数据卷所占用的空间
- `docker image prune` 清除被覆盖从而命名为 \<none\> 的的无用虚悬镜像
- `docker build` 当前目录的Dockerfile构建镜像，注意后面可以跟一个上下文路径，Dockerfile里的文件复制命令只能在这个路径的子目录下操作
- `docker save imgName | gzip > imgName-latest.tar.gz` 将镜像保存为归档文件
- `docker load -i imgName-latest.tar.gz` 从归档文件中加载镜像
- `docker save <镜像名> | bzip2 | pv | ssh <用户名>@<主机名> 'cat | docker load'` 这里的 `pv` 命令是用来显示进度条的命令，需要单独安装

## 容器命令

- `docker container ls` 列出正在运行的容器
- `docker run imgName cmd` 运行镜像里的命令(实例化成容器)，参数可以有:
    - `-t` 选项让Docker分配一个伪终端（pseudo-tty）并绑定到容器的标准输入上
    - `-i` 则让容器的标准输入保持打开
    - `-p` 指定端口映射
    - `--name` 指定容器的名字
    - `-d` 守护态运行，不输出到(stdout)即打印到宿主机上，**注意**：容器是否会长久运行，是和 docker run 指定的命令有关，和 -d 参数无关
- `docker attach` 进入容器的终端，但是一旦在这个stdin中执行exit，容器会停止，一般建议使用 `docker exec`
- `docker exec -it containerName|imgID bash|zsh` 进入容器的某一个终端
- `docker start` 通过名字或者id来启动一个已经终止的容器，等价于 `docker container start`
- `docker stop` 通过名字或者id来停止一个已经终止的容器，等价于 `docker container stop`
- `docker restart` 重新启动容器
- `docker diff container_name` 查看容器存储层的改动
- `docker commit [选项] <容器ID或容器名> [<仓库名>[:<标签>]]`
- `docker export` 导出容器，如 `docker export containerName > xxx.tar`
- `docker import` 导入容器快照文件
- `docker container rm containerName` 清除某个容器，如果容器正在执行需要加上 `-f` 参数
- `docker container prune` 清除所有处于终止状态的容器

## Dockerfile

Dockerfile一般放在项目根目录，编写完该文件后运行 `docker build` 即可生成镜像

### Dockerfile 命令

- `FROM image_name:version` 指定基础镜像，有操作系统的也有应用环境的，有一个特殊的镜像名为 `scratch` 这个镜像是虚拟的概念，并不实际存在，它表示一个空白的镜像

- `RUN shell_cmd` 执行命令，每一句 `RUN` 都会创建一层

- `COPY src_paths dst_path` 复制文件到镜像，目标路径可以是绝对路径，也可以是以 `WORKDIR somepath` 命令来指定的工作目录的相对路径，可以指定 `.dockerignore` 文件在拷贝整个目录时候忽略某些文件

- `ADD  src_paths dst_path` 同 `COPY` 命令，但是提供复制压缩包时候自动解压的功能，不用自动解压时候最好用 `COPY` 命令

- `CMD shell_cmd`, `CMD exec ["script_file", "arg1", "arg2", ...` 容器启动命令，即由本镜像创建容器时候执行的命令，执行命令或者执行文件。

- `ENTRYPOINT` 同 `CMD`，只不过可以在启动容器时候附加参数，最终会被附加到 `ENTRYPOINT` 命令上

- `ENV <key1>=<value1> <key2>=<value2>...` 指定环境变量

- `ARG` 同 `ENV` 只不过是构建的环境变量，将来容器运行的时候是无效的

- `VOLUME ["<path1>", "<path2>"...]` 定义匿名卷，容器运行时应该尽量保持容器存储层不发生写操作，动态数据(库)文件应该保存于卷(volume)中。为了防止运行时用户忘记将动态文件所保存目录挂载为卷，在Dockerfile中，我们可以事先指定某些目录挂载为匿名卷，这样在运行时如果用户不指定挂载，其应用也可以正常运行，不会向容器存储层写入大量数据。如配置 `VOLUME /data` 在启动容器的时候就可以运行 `docker run -d -v mydata:/data xxxx` 将???

- `EXPOSE port1 [port2, port3]` 声明容器开放的端口，可以在创建容器时候建立声明的端口和宿主机端口的映射，如 `docker run -p hostPort1:containerPort1 -p hostPort2:containerPort2 imageName`，或者使用参数 `-P` 将所有的暴露的端口动态分配到宿主机的端口上，也可使用 `--expose` 在创建容器时候增加容器暴露的端口

- `WORKDIR path` 指定工作目录，使用 WORKDIR 指令可以来指定工作目录（或者称为当前目录），以后各层的当前目录就被改为指定的目录，如该目录不存在，WORKDIR 会帮你建立目录

- `USER username` 和 `WORKDIR` 类似，指定之后层构建时候的用户，对于在执行脚本期间的切换用户推荐使用 `gosu` 命令而不是 `su` 或者 `sudo`

- `HEALTHCHECK` 健康检查，[详细用法](https://yeasy.gitbooks.io/docker_practice/content/image/dockerfile/healthcheck.html)

- `ONBUILD` 作为基础依赖镜像时候执行性的命令的前缀，[详细用法](https://yeasy.gitbooks.io/docker_practice/content/image/dockerfile/onbuild.html)

### Dockerfile 多阶段构建

在应用了容器技术的软件开发过程中，控制容器镜像的大小可是一件费时费力的事情。如果我们构建的镜像既是编译软件的环境，又是软件最终的运行环境，这是很难控制镜像大小的，所以常见的配置模式为：分别为软件的编译环境和运行环境提供不同的容器镜像
在 Docker 17.05 版本之前，为了将 Dockerfile 拆分可以使用，多个文件记录不同的构建步骤如：编译环境、部署环境等等，然后通过统一的 build 脚本构建每一单一的层并组合。
但是最新的做法可以在一个 Dockerfile 里根据功能模块安装多构建阶段进行分开。如下的go的示例，`FROM golang:1.9-alpine as builder` 为阶段命名，`--from=0` 指的是从上一阶段的镜像中复制文件

```dockerfile
FROM golang:1.9-alpine as builder
RUN apk --no-cache add git
WORKDIR /go/src/github.com/go/helloworld/
RUN go get -d -v github.com/go-sql-driver/mysql
COPY app.go .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .

FROM alpine:latest as prod
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=0 /go/src/github.com/go/helloworld/app .
CMD ["./app"]
```

## 数据卷

数据卷 是一个可供一个或多个容器使用的特殊目录

- `数据卷` 可以在容器之间共享和重用
- 对 `数据卷` 的更新，不会影响镜像
- `数据卷` 默认会一直存在，即使容器被删除

基本操作

```shell
docker volume create my-vol
docker volume ls
docker volume inspect my-vol
```

将 `数据卷` 挂载到容器里 `docker run -v my-vol:/wepapp`

清除数据卷，`数据卷` 是被设计用来持久化数据的，它的生命周期独立于容器，Docker 不会在容器被删除后自动删除 数据卷，并且也不存在垃圾回收这样的机制来处理没有任何容器引用的 `数据卷`

```shell
docker volume rm my-vol
# 清除无主数据卷
docker volume prune
```

也可以将本地目录(文件)挂载作为数据卷 `docker run -v /yourpath/to/volum:/wepapp`，这里宿主机的目录必须为绝对路径，默认权限为 `读写`，可以在挂载时候指定权限为只读 `docker run -v /yourpath/to/volum:/wepapp:ro`

## 网络和文件访问

### 基础

Docker中有两个概念，一个叫做 EXPOSE ，一个叫做 PUBLISH 。

- `EXPOSE` 是镜像/容器声明要暴露该端口，可以供其他容器使用。这种声明，在没有设定 `--icc=false` 的时候，实际上只是一种标注，并不强制。也就是说，没有声明 EXPOSE 的端口，其它容器也可以访问。但是当强制 `--icc=false` 的时候，那么只有 EXPOSE 的端口，其它容器才可以访问。
- `PUBLISH` 则是通过映射宿主端口，将容器的端口公开于外界，也就是说宿主之外的机器，可以通过访问宿主IP及对应的该映射端口，访问到容器对应端口，从而使用容器服务。

docker run 命令中的 -p, -P 参数，以及 docker-compose.yml 中的  ports 部分，实际上均是指 PUBLISH。

- 小写 -p 是端口映射，格式为 [宿主IP:]<宿主端口>:<容器端口>，其中宿主端口和容器端口，既可以是一个数字，也可以是一个范围，比如：1000-2000:1000-2000。对于多宿主的机器，可以指定宿主IP，不指定宿主IP时，守护所有接口。
- 大写 -P 则是自动映射，将所有定义 EXPOSE 的端口，随机映射到宿主的某个端口。

- [ ] 容器互联，swarm，k8s

### 文件传输、共享

从容器往宿主机里拷贝数据  `docker cp mycontainer：/opt/testnew/file.txt /opt/test/`
从宿主机往容器里拷贝数据`docker cp /opt/test/file.txt mycontainer：/opt/testnew/`

## 私有仓库

### 创建私有仓库的服务

通过使用官方提供的 `registry` 镜像可以方便的创建私有仓库，[文档](https://docs.docker.com/registry/)

```shell
docker run -d -p 5000:5000 --restart=always --name registry registry:2
```

几点说明

- `--restart=always` 参数指定该容器随着docker启动而启动
- 可以看到使用的是 5000 端口
- 默认的存储镜像的卷存储在容器的 `/var/lib/registry` 目录里，可以通过 `-v /yourpath/registry/:/var/lib/registry` 将其映射出来
- 使用 htpasswd 鉴权：
    - 在跑 `registry` 服务的机器上使用 `htpasswd` 生成一对用户名和密码的登录文件， 在宿主机上可以使用容器里的命令来创建，如将创建的文件保存到宿主机 `docker run --entrypoint htpasswd registry:2 -Bbn testuser testpassword > /yourpath/htpasswd`
    - 运行容器时候指定验证方式，挂载之前创建在宿主机上的登录文件的文件夹到容器内的指定的 `/auth` 目录，指定登录文件
    ```shell
    docker run -d \
    -p 5000:5000 \
    --restart=always \
    --name registry \
    -v `pwd`/auth:/auth \
    -e "REGISTRY_AUTH=htpasswd" \
    -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
    -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
    registry:2
    ```
    - 之后的就需要用 `docker login myregistrydomain.com:5000` 命令和之前设定的用户名和密码来登录 `registry` 了

### 推送到私有服务仓库

- 首先需要标记镜像，`docker tag myImg 127.0.0.1:5000/myImg`，这样会创建一个被标记过后的镜像
- 使用 `docker push 127.0.0.1:5000/myImg` 上传镜像
- 注意事项，如果使用局域网、公网来传输的话会失败，这是因为 Docker 默认不允许非 HTTPS 方式推送镜像。我们可以通过 Docker 的配置选项来取消这个限制
    - 对于 Ubuntu 14.04，编辑 `/etc/default/docker` 文件，加入 `DOCKER_OPTS="--registry-mirror=https://registry.docker-cn.com --insecure-registry=yourIp:5000"`
    - 对于使用 systemd 的系统(Ubuntu 16.04+, Debian 8+, centos 7)，编辑 `/etc/docker/daemon.json`(不存在请创建)
    ```json
    {
        "registry-mirror": [
            "https://registry.docker-cn.com"
        ],
        "insecure-registries": [
            "192.168.199.100:5000"
        ]
    }
    ```
    - 对于mac或者windows可以通过gui来设置

## 最佳实践

- 要保持容器无状态，应作为 `应用容器`
    - 容器存储层无状态，容器的存储的文件系统效率并不好，需要持久化的部分，可以使用命名卷进行持久化。由于命名卷的生存周期和容器不同，容器消亡重建，卷不会跟随消亡。所以容器可以随便删了重新run，而其挂载的卷则会保持之前的数据。
    - 服务层面的无状态，从服务层面上说，也存在无状态服务。就是说服务本身不需要写入任何文件

- 合理构建层次，构建过程中，如果某层之前构建过，而且该层未发生改变的情况下，那么 docker 就会直接使用缓存，不会重复构建。因此，合理分层，充分利用缓存，会显著加速构建速度

```dockerfile
# 以 node.js 的应用示例镜像为例，其中的复制应用和安装依赖的部分，如果都合并一起，会写成这样：
COPY . /usr/src/app
RUN npm install

# 但是，在 node.js 应用镜像示例中，则是这么写的：
COPY package.json /usr/src/app/
RUN npm install
COPY . /usr/src/app
```

## 技巧&常用命令

- Docker 的默认存储位置是 `/var/lib/docker`，如果想将一台宿主主机的 Docker 环境迁移到另外一台宿主主机可以停止 Docker 服务。可以将整个 Docker 存储文件夹复制到另外一台宿主主机，然后调整另外一台宿主主机的配置即可。

- 列出容器和其使用的卷的关系

```shell
docker inspect --format '{{.Name}} => {{with .Mounts}}{{range .}}
    {{.Name}},{{end}}{{end}}' $(docker ps -aq)
```

- 在构建容器的调试过程中，可能需要多次 build 和 run ，可以采用下面的几点
    - 在 build 的时候最好指定 `-t tag_name` 指定好镜像名，不然不能重用之前 build 的层，不能很方便的 run ，每次 build 都会参数虚悬镜像
    - 在 run 的时候可以使用 `--rm` 参数，这样在运行结束后就会自动删除容器
    - 把调试信息输出到屏幕上，别忘了 run 的时候指定参数 `-it`

- 从container连接到host主机，早期版本的对于mac来说，地址是 `docker.for.mac.localhost` ,win是 `docker.for.mac.localhost`，也就是在container里可以这样连接主机的MySQL，在docker18之后的版本统一为 `host.docker.internal`

```shell
    mysql -uroot -h docker.for.mac.localhost -p
    mysql -uroot -h host.docker.internal -p
```

- 修改docker的映射端口
    1. 停止容器
    2. 停止docker服务(systemctl stop docker)
    3. 修改这个容器的hostconfig.json文件中的端口（如果config.v2.json里面也记录了端口，也要修改）
    4. 停止docker服务(systemctl start docker)
    5. 启动容器
        - 对于Linux其/var/lib/docker/containers/[CONTAINER_ID]/目录下就有这两个文件
        - 对于mac稍微麻烦点
            1. 首先切换到 /Users/[userName]/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/目录下
            2. 然后运行 screen tty 进入docker终端
            3. 之后和Linux一样切换到相应的路径
    6. 这两个配置文件都只能用单行的json，多行会出问题

## 问题

- 绑定了宿主的文件(不是文件夹)到容器，宿主修改了文件，容器内看到的还是旧的内容，因为docker保存的是inode
- docker通过Nginx负载均衡时候，remote ip会不对，[参考](http://answ.me/post/get-real-client-ip-in-docker/)可以将 `/etc/default/docker` 添加 `DOCKER_OPTS="--userland-proxy=false"`

## 总结

### 一张图总结docker的命令

[命令总结]

{F12839}

### 参考

- [docker问答](https://blog.lab99.org/post/docker-2016-07-14-faq.html)
- [docker官方仓库](https://github.com/docker-library/docs)
