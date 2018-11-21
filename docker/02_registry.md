# docker 私有仓库搭建

搭建一个私有的 docker registry 非常简单，docker 官方提供了名为 registry/2 的镜像，直接运行该镜像就可搭建一个私有镜像服务

docker 私有仓库也提供了鉴权方式，用起来也十分方便，下面将从两个环境来讲讲，私有仓库的搭建，一个是自己玩一玩、测试开发，一个生产环境，两种方式的容器创建有所不同需要注意

## 快速上手

直接使用下面的命令即可创建一个具有鉴权功能的私有镜像仓库，

```shell
docker run -d \
  -p 5000:5000 \
  --restart=always \
  --name registry \
  -v /opt/data/registry:/var/lib/registry \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" \
  registry:2
```

几点说明

- `--restart=always` 参数指定该容器随着docker启动而启动
- 可以看到默认暴露使用的是 `5000` 端口
- 默认的存储镜像的卷存储在容器的 `/var/lib/registry` 目录里，可以通过 `-v /yourpath/registry/:/var/lib/registry` 将其映射出来，这一避免容器删除时候镜像丢失
- 这里指定了鉴权处理环境变量，即 **鉴权这一部分在运行的容器这一层次里处理**
    - 环境变量 `REGISTRY_AUTH=htpasswd` 指定了密码本管理格式
    - 环境变量 `REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd` 指定了鉴权密码本文件的路径
    - 需要注意的是，这里只是为了测试，所以没有将密码本的目录进行本机挂载，即删除容器后该容器里创建的用户信息都会丢失

### 使用步骤

创建之后，这样就会有一个 docker 仓库地址了 `http://127.0.0.1:5000` ，docker 默认只认 https ，这时候要想使用本地创建的私有仓库，就需要修改 docker 配置，为其加上非安全仓库地址，或者使用 openssl 创建一个本机的根证书和相关的证书，可以参考 docker 官方文档

#### 创建用户

使用 `htpasswd` 命令创建用户名和加密后的字符串密码字符串然后添加到，鉴权密码本文件里即可，registry 容器里带有这个命令，所以可以直接进入容器然后使用 `htpasswd -Bbn admin admin >> /auth/htpasswd` 创建一个用户名和密码都为 admin 的用户

或者在宿主机上安装 `htpasswd` 这个工具，ubuntu `sudo apt-get install apache2-utils`，在本机上创建后拷贝到容器里，或者密码本目录是挂载方式的话，直接追加到本机文件上

这样采用密码加密方式是 docker 文档推荐的 `bcrypt` 方式，更加安全，即 `-B` 选项，`htpasswd` 使用可是直接在终端中输入 `htpasswd` 查看

#### 添加非 https 的仓库地址

- Ubuntu 14.04, Debian 7 Wheezy

  对于使用 upstart 的系统而言，编辑 /etc/default/docker 文件，在其中的 DOCKER_OPTS 中增加如下内容：
  `DOCKER_OPTS="--registry-mirror=https://registry.docker-cn.com --insecure-registry=127.0.0.1:5000"`
  并重新启动服务 `sudo service docker restart`

- Ubuntu 16.04+, Debian 8+, centos 7

  对于使用 systemd 的系统，请在 /etc/docker/daemon.json 中写入如下内容（如果文件不存在请新建该文件）

  ```json
  {
    "registry-mirror": [
      "https://registry.docker-cn.com"
    ],
    "insecure-registries": [
      "127.0.0.1:5000"
    ]
  }
  ```

- 对于有 GUI 的 docker 客户端，直接在配置界面可以更改

需要注意的是 一个是 `--insecure-registry=` 一个是 `"insecure-registries": [`，而且添加之后需要重启 docker

#### 登录私有仓库

需要上传的镜像重新打 tag，如原来的 alpine 可以 `docker tag alpine:latest 127.0.0.1:5000/alpine:latest`，或者直接在 build 的时候指定带有该私有仓库对的前缀名

然后需要使用 `docker login 127.0.0.1:5000` 去登录授权，之后直接 `docker push 127.0.0.1:5000/alpine:latest` 就可以推送镜像

也可以在浏览器中打开 `http://127.0.0.1:5000/v2/` 进行登录，然后打开 `http://127.0.0.1:5000/v2/_catalog` 就可以看到该私有仓库下的镜像了

## 生产环境下

生产环境下的最大不同就是，使用 80/443 端口创建虚拟主机，使用 Nginx 来代理请求，并将鉴权这一部分移到 Nginx 上来，**这时候创建私有仓库的方式就和之前的不一样了，也就是不能加上鉴权的配置信息**，同时也将 https 处理移到 Nginx 来处理，这样会更方便，一个容器创建示例如下

```shell
docker run -d \
  -p 5000:5000 \
  --restart=always \
  --name registry \
  -v /opt/data/registry:/var/lib/registry \
  registry:2
```

### 配置反向代理

关于 Nginx 去代理私有仓库容器，[官方文档](https://docs.docker.com/registry/recipes/nginx/)给出的是 docker-compose 的方法，这里需要注意的是，nginx:alpine 镜像是添加了对 `bcrypt` 的支持，而默认 Nginx 是不支持这种方式的，下面的设置会以非容器方式去部署 Nginx 所以不能使用 `bcrypt`，创建一个 Nginx 虚拟容器配置

```conf
server {
        listen      443;
        server_name    docker.xxxx.com;

        ssl on;
        ssl_certificate   cert/xxxx.pem;
        ssl_certificate_key  cert/xxxx.key;
        ssl_session_timeout 5m;
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers on;

        # for docker registry proxy setting
        chunked_transfer_encoding on;
        client_max_body_size 0;

        location /v2/ {
            auth_basic "Registry realm";

            auth_basic_user_file /etc/nginx/.htpasswd;
            add_header 'Docker-Distribution-Api-Version' 'registry/2.0' always;

            proxy_pass                          http://127.0.0.1:5000;
            proxy_set_header  Host              $http_host;   # required for docker client's sake
            proxy_set_header  X-Real-IP         $remote_addr; # pass on real client's IP
            proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header  X-Forwarded-Proto $scheme;
            proxy_read_timeout                  900;
        }
}
```

https 证书可以使用自创建的，但是其他机器要用的话需要添加根证书，比较麻烦，可以直接使用免费的 [letsencrypt](https://letsencrypt.org/getting-started/)

### 添加用户

和 [上文](####创建用户) 一样只不过不能使用 `bcrypt` ，密码本文件在宿主机上，由 Nginx 配置指定，如上面的配置，为 `/etc/nginx/.htpasswd` 文件；

创建的话可以使用：

- 本地安装或者 registry 容器里的 htpasswd 命令：`htpasswd -bn admin admin >> /etc/nginx/.htpasswd` 来创建
- 或者 OpenSSL ，`printf "admin:$(openssl passwd -apr1 admin)\n" >> /etc/nginx/.htpasswd`

然后 `sudo nginx -s reload` 一下就可以了，如果不能访问的话，可以查看 Nginx 的错误日志看看具体问题所在

### 填坑

- Ubuntu 14.04 的 Nginx 使用 Ubuntu 源其版本会比较低，不支持 add_header 后面的 always 参数，可以使用下面的命令来更新 Nginx

```shell
sudo add-apt-repository ppa:nginx/stable
sudo apt-get update
sudo apt-get install nginx
```

## Registry api

registry提供了一系列的访问文档，[文档](https://docs.docker.com/registry/spec/api/)，下面列举一二

登录

- 首先，registry 需要鉴权，验证方式由上面的 Nginx 配置可以知道，为 [basic auth](https://en.wikipedia.org/wiki/Basic_access_authentication)，即在请求的 header 有如下的内容 `Authorization: Basic XXX` 其中的 XXX 即为根据用户名和密码的 base64 编码格式为 user@passworld，所以这种方式在传输过程中比较危险，非 https 环境下建议不要使用
- 客户端如果用浏览器，访问 `https://docker.hashworld.top/v2/` 时候会提示输入用户名和密码，之后浏览器会保存，并且在其他请求下也会带入该验证信息头
- 如果使用 postman 可以通过在在 collection 的 auth 配置里加入 base auth，如使用 curl 可使用 `curl -u uname:passswd https:/docker.hashworld.top/v2/`

常用

- `https://docker.hashworld.top/v2/_catalog` 获取 docker 的所以镜像名
- `https://docker.hashworld.top/v2/imgName/tags/list/` 获取镜像名为 imgName 的所以 tag
- `https://docker.hashworld.top/v2/imgName/manifests/latest/` 获取镜像名为 imgName ，tag 为 latest 的信息

## Harbor

[Harbor](https://github.com/vmware/harbor)是一个开源的企业级私有仓库

这里使用这样的安装方式：nginx (host,ssl) -> harbor-nginx(non-ssl) -> harbor

即使用宿主机的 Nginx 进行内部容器 harbor-nginx 的代理，宿主机使用 Nginx https，harbor-nginx 使用 http。

- 首先确保安装 Python、Docker、Docker-compose 等组件
- 在[发布界面](https://github.com/goharbor/harbor/releases)下载 harbor 最新的离线安装包(大概800m，使用代理会更快)，如 v1.5.4 版本

  `$ wget https://storage.googleapis.com/harbor-releases/harbor-offline-installer-v1.5.4.tgz`

- 解压 harbor

  `$ tar xvf harbor-offline-installer-<version>.tgz`

- 配置 harbor/harbor.cfg，这里给出了当前应用部署环境需要更改的配置项，和几个重要项

```cfg
  #这里的 hostname 安装注释应该是一个公网可访问的域名或者ip，是为了方便后面的配置生成，后面会对这个域名进行搜索然后进行定制
  hostname = docker.hashworld.top:5080
  #由于 harbor-nginx 使用在内部，直接使用 http
  ui_url_protocol = http

  #这个地方不能修改，不然会出奇怪的问题
  secretkey_path =  /data
  #修改初始化的管理员密码，之后可以在 harbor ui 里直接进行修改
  harbor_admin_password = Harbor12345
  #关闭自主注册，如果开启，需要配置 email 的 stmp 的配置
  self_registration = off
```

- 修改 `docker-compose.yml` 的 proxy 服务的端口暴露，避免端口冲突，如下，其中 80 修改为上面的 harbor.cfg 里的 hostname 端口 5080 保持一致

```yml
  ports:
    - 5080:80
    - 5443:443
```

- 由于使用外部 Nginx 代理，需要修改下 harbor-Nginx 的配置文件，[参见](https://github.com/goharbor/harbor/blob/master/docs/installation_guide.md#troubleshooting)，也就是注释去掉 `common/templates/nginx/nginx.http.conf` 里面的所有的下面的内容

```conf
  proxy_set_header X-Forwarded-Proto $scheme;
```

- 启动安装

  `$ sudo ./install`

- 安装不出问题的话，harbor 的服务应该使用 docker-compose 都启动完毕，这一部时候，`install.sh` 会调用 `prepare.sh` 根据配置文件 `harbor.cfg` 在  common 文件夹下生成一些用于镜像的配置，这里需要根据部署环境修改，带有 `docker.hashworld.top` 的部分的配置

```shell
  $ cd harbor
  $ sudo grep -r "docker.hashworld.top"
  config/adminserver/env:EXT_ENDPOINT=http://docker.hashworld.top:5080
  config/registry/config.yml:    realm: http://docker.hashworld.top:5080/service/token
  harbor.cfg:hostname = docker.hashworld.top:5080
```

修改上述的 env 和 config.yml ，修改 `http://docker.hashworld.top:5080` 为 `https://docker.hashworld.top`。env 文件是为了让匿名访问时候共用仓库的镜像复制的拉取命令是正确的。config.yml 文件是修复 Registry 容器能够从公网使用 harbor 的用户验证系统，也就是修复 `docker login` 错误

- 添加宿主机 Nginx 为 harbor 的 https 代理，参考配置如下

```conf
  server {
        listen      443;
        server_name    docker.hashworld.top;

        ssl on;
        ssl_certificate   cert/yourstuff.pem;
        ssl_certificate_key  cert/yourstuff.key;
        ssl_session_timeout 5m;
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers on;

        # for docker registry proxy setting
        chunked_transfer_encoding on;
        client_max_body_size 0;

        location / {
            proxy_pass                          http://127.0.0.1:5080;
            proxy_set_header  Host              $http_host;   # required for docker client's sake
            proxy_set_header  X-Real-IP         $remote_addr; # pass on real client's IP
            proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header  X-Forwarded-Proto $scheme;

        }
  }
```

这里的 `http://127.0.0.1:5080;` 是因为，之前修改的 docker-compose.yml 文件的缘故，直接使用本地的连接方便一些(外网的端口访问权限有可能没有开启)

- 使用时候需要注意的是，harbor 使用第二级信息作为项目名，如默认的 library，需要将镜像 tag 成这样的格式 `docker.hashworld.top/library/imageName:imageTag`

### refer

- [use nginx](https://github.com/goharbor/harbor/issues/3114)
- [Installation and Configuration Guide](https://github.com/goharbor/harbor/blob/master/docs/installation_guide.md)

## 参考

- [配置密码](https://docs.docker.com/registry/deploying/#native-basic-auth)
- [使用Nginx代理](https://docs.docker.com/registry/recipes/nginx/#setting-things-up)
- [问题参考](https://github.com/docker/distribution/issues/655)
- [博客参考](https://www.jianshu.com/p/fc544e27b507)
