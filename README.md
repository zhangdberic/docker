# docker 

## docker安装

**基于centos7安装docker**

### 设置yum源

mkdir -p /bak/etc/yum.repos.d/

mv /etc/yum.repos.d/*.repo /bak/etc/yum.repos.d/

下载Centos版本对应的软件仓库文件

cd /etc/yum.repos.d

wget http://mirrors.163.com/.help/CentOS7-Base-163.repo -O CentOS-Base.repo

如果没有wget，则使用：

curl http://mirrors.163.com/.help/CentOS7-Base-163.repo > CentOS-Base.repo

安装epel源

curl http://mirrors.aliyun.com/repo/epel-7.repo > epel-7.repo

运行以下命令生成缓存

yum clean all

yum makecache

### 安装docker依赖的软件

yum install -y yum-utils device-mapper-persistent-data lvm2

### 安装docker的yum源

yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

### 禁止docker-ce-edge和docker-ce-test

yum-config-manager --disable docker-ce-edge

yum-config-manager --disable docker-ce-test

### 更新yum

yum makecache fast

### 安装docker-ce

**列出docker-ce安装包**

yum list docker-ce.x86_64 --showduplicates|grep stable |sort -r

**命令格式：yum install docker-ce-版本**

yum install docker-ce-19.03.4-3.el7

### 启动docker

systemctl start docker

### 验证安装docker成功

docker run hello-world

### 配置docker镜像下载阿里云加速

mkdir -p /etc/docker

tee /etc/docker/daemon.json <<-'EOF'

{

 "registry-mirrors": ["https://7qfuu9af.mirror.aliyuncs.com"]

}

EOF

systemctl daemon-reload

systemctl restart docker

## 安装私有仓库(registry2)





## docker命令

### docker inspect xxx

查看docker镜像实例运行情况，命令：

docker inspect 容器名和容器ID

例如：docker inspect redis1

查看运行的ip地址

docker inspect redis1 | grep IPAddress



## docker 镜像

### redis镜像

**准备工作**

搜索redis镜像

```
docker search redis
```

下载redis镜像

```shell
docker pull redis
```

**创建redis docker挂载目录**

注意：下面的**redis1**为这个redis docker实例的名字,一个宿主机上允许多个redis docker,因此要区别开。

下面的配置会依赖这个redis1的docker实例名，如果需要部署多个可以替换这个redis1字样为redis2、redis3...

```shell
mkdir -p /data/redis1/data
mkdir -p /data/redis1/config
```

**下载redis.conf配置文件**

```shell
cd /data/redis1/config
wget https://github.com/zhangdberic/docker/blob/master/redis.conf -O /data/redis1/config/redis.conf
```

vi /data/redis1/config/redis.conf

```properties
protected-mode no # 保护模式设置为no
bind 0.0.0.0    # docker环境应该使用0.0.0.0绑定,安全有宿主的防火墙控制
daemonize yes   # 基于后台允许
databases 100   # 允许的最大数据库数量
#save 900 1     # 禁止持久化
#save 300 10
#save 60 10000
save ""
maxmemory 5368709120   # 允许使用的最大内存空间
maxmemory-policy volatile-lru   
# 当redis中内存数据达到maxmemory时,触发"清除策略"，volatile-lru  ->对"过期集合"中的数据采取LRU(近期最少使用)算法.如果对key使用"expire"指令指定了过期时间,那么此key将会被添加到"过期集合"中。将已经过期/LRU的数据优先移除.如果"过期集合"中全部移除仍不能满足内存需求,将OOM.

requirepass xxxxxx   #秘钥

logfile /data/redis1/data/redis.log # 指定log文件的位置,启动docker的时候,会把/data挂载到宿主目录

注意：上面的配置修改后，最好先修改daemonize no，启动一次redis，看是否有什么警告，如果没有再修改为yes。
```

**运行redis**

```shell
docker run --name redis1 -p 6379:6379 \
-v /data/redis1/config/redis.conf:/etc/redis/redis.conf -v /data/redis1/data:/data \
--privileged --sysctl net.core.somaxconn=10240 \
-d redis redis-server /etc/redis/redis.conf
```

**查看redis日志**

```shell
tail -f /data/redis1/data/redis.log
```

**telnet测试远程连接**

```
telnet 192.168.1.250 6379
```

**访问redis容器**

```shell
docker exec -it redis1 /bin/bash
redis-cli
```

**redis-cli访问**

```shell
docker exec -it redis1 redis-cli -h 127.0.0.1 -p 6379
```



### Rabbitmq镜像

搜索和下载rabbitmq镜像

```
docker search rabbit
```

```
docker pull rabbitmq
```

运行rabbitmq镜像

```shell
docker run -d --name rabbitmq1 -p 5672:5672 -p 15672:15672 -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=Rabbitmq-401 docker.io/rabbitmq
```



### GitLab镜像

**搜索Gitlab镜像**

```
 docker search gitlab
```

**下载Gitlab镜像**

```
docker pull gitlab/gitlab-ce
```

**创建Gitlab挂载目录**，并分别创建子目录config,logs,data

```shell
mkdir -p /data/gitlab/config
mkdir -p /data/gitlab/data
mkdir -p /data/gitlab/logs
```

**运行gitlab**

```shell
docker run --name='gitlab' -d \
    --publish 1443:443 --publish 7001:80 --publish 7002:22 \
    --volume /data/gitlab/config:/etc/gitlab \
    --volume /data/gitlab/logs:/var/log/gitlab \
    --volume /data/gitlab/data:/var/opt/gitlab \
    gitlab/gitlab-ce
```

**登录Gitlab** 

http://宿主机ip:7001 

成功的话需要修改root账号的密码，随意设置即可。密码修改成功后，系统进入登录/注册页面

**配置gitlab**

vi /data/gitlab/config/gitlab.rb

官方文档：https://docs.gitlab.com/omnibus/settings/configuration.html

**备份**

只需备份/data/gitlab目录就可以了。