# docker 化部署

## 镜像更新问题

### 文件依赖

- conf、data、log问题，[ref](http://jm.taobao.org/2016/06/08/docker-maintenance/)
- 私有外部服务连接配置（数据库、oss、Redis...）
- 内部私有、敏感文件，也就是.gitignore 里面的文件
    - 日志文件
    - 一些管理员用户上传的文件如 src/concealfiles/cheaterfiles/
    - 部分用户上传的东西？src/uploadfiles/

最后可以通过volume将这些东西从宿主机上挂载，所以工程的目录结构需要变化，将需要从宿主机挂载的部分集中起来

- 这样之后，镜像是否能够提供给后续开发的快速环境搭建：
    - 本地开发，该镜像是否可以被从私有hub上拉下来
    - 测试服开发，运行镜像时候可以指定配置目录，可以运行多个目录从而运行多个docker

### 私有Docker hub搭建和镜像问题

- 镜像应该在sandbox上发布，是否存储
- docker 命令
    - commit
    - save
    - export

另外，Docker 不是虚拟机，其文件系统是 Union FS，分层式存储，每一次 commit 都会建立一层，上一层的文件并不会因为 rm 而删除，只是在当前层标记为删除而看不到了而已，每次 docker pull 的时候，那些不必要的文件都会如影随形，所得到的镜像也必然臃肿不堪。而且，随着文件层数的增加，不仅仅镜像更臃肿，其运行时性能也必然会受到影响。这一切都违背了 Docker 的最佳实践。

使用 commit 的场合是一些特殊环境，比如入侵后保存现场等等，这个命令不应该成为定制镜像的标准做法。所以，请用 Dockerfile 定制镜像。

### 数据库迁移问题

- 多分支开发下的数据迁移问题（是否记录迁移文件），迁移文件是否影响程序的无状态
    - [讨论](https://stackoverflow.com/questions/50274624/django-migrations-and-docker-containers)
    - [记录迁移文件](https://www.algotech.solutions/blog/python/django-migrations-and-how-to-manage-conflicts/)
    - [中文版](https://hexiangyu.me/2016/06/20/django-migrations-and-how-to-manage-conflicts/)
- migrate文件和数据库的django_migrations表的对应
- sandbox的MySQL容器的备份

比如官方 mysql 镜像中，可以把初始化的 .sql 脚本文件在 Dockerfile 中 COPY 至 /docker-entrypoint-initdb.d/ 目录中，在容器第一次运行的时候，如果所挂载的卷是空的，那么就会依次执行该目录中的文件，从而完成数据库初始化、导入等功能。

### 日志问题

- 比如前端 nginx，它不需要写入任何文件（日志走Docker日志驱动）
- 关于日志收集，Docker 内置了很多日志驱动，可以通过类似于 fluentd, syslog 这类服务收集日志。无论是 Docker 引擎，还是容器，都可以使用日志驱动

```dockerfile
# forward request and error logs to docker log collector
RUN ln -sf /dev/stdout /var/log/nginx/access.log \
	&& ln -sf /dev/stderr /var/log/nginx/error.log
```

- docker volume 自然支持很多分布式存储的驱动，比如 flocker、glusterfs、ceph、ipfs 等等。常用的插件列表可以参考[官方文档](https://docs.docker.com/engine/extend/legacy_plugins/#/volume-plugins)

## TODO

升级回滚：系统采用可回滚的双分区模式，活动的分区通过只读方式挂载，另外一个分区用来自动更新, 通过切换系统分区即可实现快速升级，升级出现问题时，可以快速切换回原来的分区保证系统可用；

- [ ] `.dockerignore` 文件应该忽略git版本记录？

## hashworld-core 步骤

- 将顶层文件夹命名为 hashworld-core，所以 django 的层次应该为 `hashworld-core/src/hashworld`

### 恢复 git 记录

```shell
# cd hashworld-xxx
git init
git remote add origin https://github.com/McGrady00H/hashworld-core.git
git fetch origin master
git reset --hard origin/master
```

### 修改代码私有配置结构

- 修改移动 `verfiy.ini` 和 `local.py` 到新建的目录 hashworld/settings/local/
- 修改/添加 local.py 文件的 `VERIFY_SETTINGS_FILE = "config/verify.ini"` 的配置路径为 `hashworld/settings/local/verify.ini`
- 新建 `hashworld/settings/local/__init__.py` 内容为 `from local.py import *`
- ~~修改 `.gitignore` 增加 `hashworld/settings/local/*`~~
    - [x] 将其提交到版本记录里
- ~~删除 config/supervisor/redis.conf~~
- ~~修改 supervisord.conf 文件的 `logfile=/tmp/supervisord.log` 到logging目录方便统一挂载日志目录 `../logging/supervisord.log`~~
- ~~增加req python-dateutil==2.7.3~~
- 修改uWSGI.ini，修改几个路径设置，示例如下

```ini
...
chdir = ../
logto = ../logging/uwsgi.log
...
```

### 构建镜像

- 复制相关的 `Dockerfile`、`.dockerignore` 到项目的 `src` 目录下
    - [ ] 将其提交到版本记录里，用于持续构建镜像
- 构建镜像，切换到项目的 src目录下，使用命令 `docker build -f Dockerfile . -t docker.hashworld.top/hashworld-core`

## nginx-backend 构建

- `docker build -f Dockerfile.nginx . -t docker.hashworld.top/nginx-backend`

## hashworld-chatroom 构建

- 复制相关的 `Dockerfil.chatroom`、`.dockerignore` 到项目的 `src` 目录下
- 修改 supervisord.conf 文件的 `logfile=/tmp/supervisord.log` 到logging目录方便统一挂载日志目录 `../logging/supervisord.log`
- ~~修改 `requirements.txt`，增加 `redis==2.10.6`~~

## compose 文件

把外部镜像的连接信息用 env 文件？然后放在,(rabbitmq 现在不是)

## 初始化

可能需要初始化数据库

./manage.py migrate && ./manage.py loaddata init.json && ./manage.py inituser && ./manage.py initland

## hashworld-cluster

1. 做不到一键安装，配置local非常麻烦。能否配置一些默认的，然后线上把关键地方改下就行了。
2. Makefile里面build的路径应该有问题，应该再往下一层到src或者services，但是目前chatroom和core并不统一，就分开来弄吧。
3. Makefile里面没有做到make migrations
4. Dockerfile里面没有 migrate

2018-8-30

- [ ] hashworld-chatroom 的supervisord配置日志目录还没有改成相对路径
- [ ] hashworld-core 的 config 目录下没有 uWSGI配置文件，而且该配置文件需要保证是相对路径的
- [ ] Dockerfile 和 .dockerignore 文件放入版本库
- [ ] rabbitmq 是在 ini 文件里设置的，不能使用 env 文件统一有设定(local.py)
- [ ] rpc 配置在 chatroom 里是 local.py 但是在 core 里面是 verify.ini 文件，不好统一设定
- [ ] 容器对配置的依赖太严重，想直接借助容器 makemigration 需要一堆配置
- [ ] 如果要在开始就 migrate 的话，容器的需要等待 MySQL 启动，需要建立一个等待服务启动的脚本 [例子](https://docs.docker.com/compose/startup-order/)

2018-8-31

- 启动时候会发现的问题（由于没有严格启动顺序）：
    - celery.beat 有时候会一直重启
    - 由于有migrate，所以需要等待数据库启动
- Dockerfile 里面不使用 CMD 而是使用 ENTRYPOINT 去执行脚本，最新的 Dockerfile 都改成了这个方式，这样扩展性较好，现在使用的是 `../scripts/startup.sh` 脚本(相对config目录)，所以需要将该文件放入两个工程的版本记录里(注意要有执行权限)，现在脚本里预设了等待的命令，可以直接在 docker-compose 文件里指定需要等待连接，而不需要修改 Dockerfile 或者脚本，示例如下
    ```yml
    service:
    version: "3"
      web:
        image: xxx
        command: --wait mysql:3306 rabbitmq:5672 redis-1:6379
    ```
- [ ] 现在的 hashworld-core, hashworld-chatroom, nginx-backend 都有相应的 alpine 镜像，标签为 alpine 的就是，三个镜像其总体积 500MB 左右，轻度测试没有发现问题
- [ ] `../scripts/startup.sh` 脚本现在还没有启动时候完成 migrate 动作，感觉在容器启动时候 migrate 不太好，应该和 makemigrations 那样随用随删地启动一个(一些)镜像当中应用容器去做这个事情
- [ ] docker compose up 多次 hashworld-cluster 可能会有问题？up起来的容器名前缀是 compose 文件的上一级文件夹名，应该有另外指定的方式，看有没有启动多个 compose 这样的需求
