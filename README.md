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

去掉前面的3:，取后面的19.03.8-3.el7为版本号

yum install docker-ce-19.03.8-3.el7

```
docker-ce.x86_64            3:19.03.8-3.el7                     docker-ce-stable
docker-ce.x86_64            3:19.03.7-3.el7                     docker-ce-stable
```

### 配置docker镜像下载阿里云加速

mkdir -p /etc/docker

tee /etc/docker/daemon.json <<-'EOF'

{

 "registry-mirrors": ["https://7qfuu9af.mirror.aliyuncs.com"]

}

EOF

### docker非root用户环境运行

#### 创建docker用户

useradd docker -g docker

root用户 passwd docker

#### sudo docker用户

visudo

docker用户加入到sudo中

#### 启动docker服务

su - docker

sudo systemctl start docker

如果普通用户执行docker命令，如果提示get …… dial unix /var/run/docker.sock权限不够，则修改/var/run/docker.sock权限
使用root用户执行如下命令，即可

```
sudo chmod a+rw /var/run/docker.sock
```

### 验证安装docker成功

docker run hello-world

## 配置防火墙控制发布端口访问

默认情况下非host模式，使用publish参数发布docker容器端口，发布出去的端口不受firewall-cmd限制，其使用iptables来优先转发端口数据，不受控，这在很多情况下有安全问题。这里我们加入了配置 --iptables=false，其还可以提供端口转发，但访问受到了firewall-cmd限制，例如：设置后 -p 6379:6379 的docker容器发布操作，如果你没有使用firewall-cmd开启6379，则外界无法访问。

vi /lib/systemd/system/docker.service

查找:

```
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```

在其后加入：--iptables=false

```
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --iptables=false
```

重启docker

```
sudo systemctl daemon-reload
sudo systemctl restart docker
```



## 安装私有仓库(registry2)

### 编辑/etc/hosts文件

定义一个虚拟的网址：

vi /etc/hosts

192.168.5.76	dockerdongyuit.cn

这里的192.168.5.76为本机ip地址

### 安装证书(支持ssl)

mkdir -p /home/docker/registry2/certs/

 openssl req -newkey rsa:2048 -nodes -sha256 -keyout /home/docker/registry2/certs/domain.key -x509 -days 3650 -out /home/docker/registry2/certs/domain.crt

```
Country Name (2 letter code) [XX]:CN

State or Province Name (full name) []:liaoning

Locality Name (eg, city) [Default City]:shenyang

Organization Name (eg, company) [Default Company Ltd]:dongyuit

Organizational Unit Name (eg, section) []:dev

Common Name (eg, your name or your server's hostname) []:dockerdongyuit.cn
```

 生成两个文件:ll /home/docker/registry2/certs/

-rw-r--r--. 1 root root 2065 11月  6 15:43 domain.crt

-rw-r--r--. 1 root root 3268 11月  6 15:43 domain.key

### 创建密码文件

mkdir -p /home/docker/registry2/auth/

docker run --entrypoint htpasswd registry:2 -Bbn heige 12345678 >>    /home/docker/registry2/auth/htpasswd

这里的heige为用户名,12345678为密码.

### 创建数据存放目录

mkdir /home/docker/registry2/data

### 启动Secure Registry

#### 非集群模式安装docker私有仓库

```shell
  docker run -d -p 5000:5000 --restart=always --name registry \
  -v /home/docker/registry2/auth:/auth \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" \
  -v /home/docker/registry2/data:/var/lib/registry \
  -v /home/docker/registry2/certs:/certs \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  registry:2 
```

解释：

-d 后台运行

--restart=always 重启容器自启动

--name 容器实例名

-v /home/docker/registry2/auth:/auth \ 映射宿主/home/docker/registry2/auth目录到容器的/auth目录

如下三个指定了认证方式和密码文件位置

-e "REGISTRY_AUTH=htpasswd" \

-e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \

-e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" \

-v /home/docker/registry2/data:/var/lib/registry \ 映射宿主/home/docker/registry2/data/目录到容器的/var/lib/registry（存储仓库）

-v /home/docker/registry2/certs:/certs \ 映射宿主/registry2/certs目录到容器的/certs目录

设置TLS认证文件位置

-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \

-e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \

启动的docker镜像

registry:2

#### docker swarm mode模式安装docker私有仓库

注意：--constraint node.labels.test-host:test2 配置，通过label指定了registry部署在集群中的某个机器，例如：本配置指定了部署在test-host:test2的label机器上，因为这个test-host:test2只在docker2的机器上配置了，因此registry2会被swarm部署到docker2上。

```shell
docker service create --name registry --publish 5000:5000 \
--mount type=bind,src=/home/docker/registry2/auth,dst=/auth \
-e "REGISTRY_AUTH=htpasswd" \
-e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
-e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" \
--mount type=bind,src=/home/docker/registry2/data,dst=/var/lib/registry \
--mount type=bind,src=/home/docker/registry2/certs,dst=/certs \
-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
-e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
--constraint node.labels.test-host==test2 \
registry:2 
```

#### 

### 测试发布到私有仓库(certificate signed by unknow authority)

如下的操作其实都应是私有仓库客户端操作，以下是为了测试方便，私有仓库和客户端docker在一台机器上模拟测试了。如果私有仓库就一台单独的机器，则没有必要下面的操作，其应在docker客户端(pull和push)的机器上完成。

重新制作一个hello-word的tag

docker tag hello-world dockerdongyuit.cn:5000/hello-world

发布到仓库

docker push dockerdongyuit.cn:5000/hello-world

报错：

```
The push refers to repository [dyit.com:5000/hello-word]
Get https://dyit.com:5000/v2/: x509: certificate signed by unknow authority
```

push失败了！从错误日志来看，docker client认为server传输过来的证书的签署方是一个unknown authority（未知的CA），因此验证失败。我们需要让docker client安装我们的CA证书：

sudo mkdir -p /etc/docker/certs.d/dockerdongyuit.cn:5000

sudo cp /home/docker/registry2/certs/domain.crt /etc/docker/certs.d/dockerdongyuit.cn:5000/ca.crt

重试发布到仓库，提示(no basic auth credentials)

docker push dockerdongyuit.cn:5000/hello-world

### 测试发布到私有仓库(no basic auth credentials)

docker push dockerdongyuit.cn:5000/hello-world

```
The push refers to repository [dyit.com:5000/hello-world]
af0b15c8625b: Preparing 
no basic auth credentials
```

push失败了！提示没有basic auth认证。

**认证登录**

docker login  dockerdongyuit.cn:5000

正确输入用户名和密码

```
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

**重试发布到仓库，成功**

docker push dockerdongyuit.cn:5000/hello-world

```
The push refers to repository [dockerdongyuit.cn:5000/hello-world]
9c27e219663c: Pushed 
latest: digest: sha256:90659bf80b44ce6be8234e6ff90a1ac34acbeb826903b02cfa0da11c82cbc042 size: 525
```



### 客户端访问私有仓库(客户端)

我们换其他机器试试访问这个secure registry。根据之前的要求，我们照猫画虎的修改一下hosts文件，安装ca.cert，docker login登录。之后尝试从registry pull image.

**编辑/etc/hosts文件**

定义一个虚拟的网址：

vi /etc/hosts

192.168.5.76	dockerdongyuit.cn

 **拷贝crt文件**

sudo mkdir -p /etc/docker/certs.d/dockerdongyuit.cn:5000

sudo cp /home/docker/registry2/certs/domain.crt /etc/docker/certs.d/dockerdongyuit.cn:5000/ca.crt

注意docker私有仓库上的domain.crt文件

**认证登录**

docker login  dockerdongyuit.cn:5000

**从私有仓库中拉去hello-world到本地**

docker login  dockerdongyuit.cn:5000

```
Pull Completed
```

### 私有仓库操作

#### 列出仓库镜像

https://dockerdongyuit.cn:5000/v2/_catalog

输入用户名和密码，heige，12345678

#### 查看某个镜像

https://dockerdongyuit.cn:5000/v2/<name>/tags/list

输入用户名和密码，heige，12345678

例如：

```
https://dockerdongyuit.cn:5000/v2/centos/tags/list
```

## windows客户端(eclipse maven发布docker)

### 安装证书

**修改host文件，加入dockerdongyuit.cn**

192.168.5.76	dockerdongyuit.cn

**安装证书**

从私有仓库服务器上拷贝/home/docker/registry2/certs/domain.crt到windows桌面(sz /registry2/certs/domain.crt)，双击这个domain.crt文件安装证书，将其安装到“安装证书->将所有的证书存放到下列存储(P)->受信任的根证书颁发机构”下。

https://dockerdongyuit.cn:5000/v2/_catalog

### 仓库服务器端开启2375端口

防护墙允许访问2375端口

firewall-cmd --permanent --add-port=2375/tcp

firewall-cmd --reload

 

vi /usr/lib/systemd/system/docker.service

查找到ExecStart，替换为如下：

ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock

```
#ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock
```

sudo systemctl daemon-reload // 加载docker守护线程

sudo systemctl restart docker // 重启docker

查看2375端口是否开启。

netstat -ntlp|grep 2375

### 设置windows环境变量DOCKER_HOST

windows加入系统环境变量，名称：DOCKER_HOST，值：tcp://dockerdongyuit.cn:2375

注意设置后，响应的cmd shell窗口和eclipse都要重新启动，否则无法正确的获取到这个环境变量。

### Eclipse的maven设置

1. 设置对应私有仓库的登录用户和密码

Eclipse下设置settings.xml文件，添加server，如下：

```xml
  <servers>
	<server>
		<id>dockerdongyuit.cn:5000</id>
		<username>heige</username>
		<password>12345678</password>
		<configuration>
			<email>909933699@qq.com</email>
		</configuration>
	</server>    
  </servers>
```

注意必须有email配置项目，否则报错：

2. 编辑项目的pom.xml文件

```xml
	<build>
		<plugins>
			<!-- 引入spring-boot的maven插件 -->
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
				<configuration>
					<fork>true</fork>
				</configuration>
			</plugin>
			<!-- 引入docker打包maven插件 -->
			<plugin>
				<groupId>com.spotify</groupId>
				<artifactId>docker-maven-plugin</artifactId>
				<version>0.4.13</version>
				<configuration>
					<!-- 访问 docker api 端口 -->
					<dockerHost>http://dockerdongyuit.cn:2375</dockerHost>
					<!-- dyit.com:5000为仓库地址 -->					<imageName>dockerdongyuit.cn:5000/dy/${project.name}:${project.version}</imageName>
					<!-- 使用的基础镜像,生成Dockerfile的FROM指令 -->
					<baseImage>base/java:1.8</baseImage>
					<!-- docker启动后执行的命令,生成Dockerfile的ENTRYPOINT指令 -->
					<entryPoint>["sh","-c","exec java $JAVA_OPTS -jar /${project.build.finalName}.jar $APP_ENV"]</entryPoint>
					<resources>
						<resource>
							<targetPath>/</targetPath>
							<directory>${project.build.directory}</directory>
							<!--添加到镜像的文件,生成Dockerfile的ADD指令  -->
							<include>${project.build.finalName}.jar</include>
						</resource>
					</resources>
					<!-- 覆盖相同tag的镜像(支持重复发布) -->
					<forceTags>true</forceTags>
					<!-- Basic auth 的配置，见settings.xml配置  -->
					<serverId>dockerdongyuit.cn:5000</serverId>
				</configuration>
			</plugin>						
		</plugins>
	</build>
```

执行打包推送的maven指令: mvn -e clean package docker:build -DpushImage



解析：

注意：这个docker的maven的插件配置，依赖于base/java:1.8的镜像。entryPoint其基于函数模式运行(sh -c exec xxxx)，其会在运行的时候替换所有的变量。具体见docker entryPoint章节。

 对应docker的运行命令如下：

run -itd --cap-add=SYS_PTRACE --name sc-config1 --net host -e JAVA_OPTS="-Xms1g -Xmx1g -XX:+UseParNewGC -XX:+UseConcMarkSweepGC" -e APP_ENV="--spring.profiles.active=dev --server.port=6000" dockerdongyuit.cn:5000/sc/sc-config:1.0.1 /bin/bash

解释上面命令：

--cap-add=SYS_PTRACE 加入跟踪允许容器内执行jmap -heap {java进程}，否则报错。

--net host 基于主机网络模式，保证服务对eureka注册的正确性。如果使用默认的网桥模式则注册的ip是172.17.0.x地址，外界无法访问。而且注册的主机名也会随着容器重启而不断的变化。

-e JAVA_OPTS="-Xms1g -Xmx1g -XX:+UseParNewGC -XX:+UseConcMarkSweepGC" 设置java启动参数变量值。

-e APP_ENV="--spring.profiles.active=dev --server.port=6000" 设置应用的环境变量值。

 进入docker后使用ps -ef|grep java结果如下：

java -Xms1g -Xmx1g -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -jar /sc-config-1.0.1.jar --spring.profiles.active=dev --server.port=6000



## docker命令

### docker服务操作命令

##### 重启docker服务

重启docker

```
sudo systemctl daemon-reload
sudo systemctl restart docker
```

#### docker info

查看docker系统的信息，例如：docker版本、使用存储情况等。

### docker容器操作命令

#### docker开机自动启动

```
--restart=always 
```

#### docker inspect 查看容器运行情况

docker inspect  xxx

查看docker镜像情况，命令：

docker inspect 容器名和容器ID

查看底层运行的jvm版本、容器内运行软件的版本等、软件默认的配置信息等。

例如：docker inspect redis1

查看运行的ip地址

docker inspect redis1 | grep IPAddress

查看运行的user

docker inspect jenkins/jenkins | grep User

#### docker stop 停止容器

docker stop xxx

#### docker stop -t sss 关闭容器的限时(秒)

参数 -t：关闭容器的限时(秒)，如果超时未能关闭则用kill强制关闭，默认值10s，这个时间用于容器的自己保存状态
docker stop -t=60 xxx

#### docker kill 强制直接关闭容器

docker kill xxx

#### docker restart 重启容器

docker restart xxx

#### docker exec 进入容器

docker exec -it xxx /bin/bash

#### docker exec 运行容器内命令

docker exec -it xxx 容器内命令





## docker 镜像

### 查找镜像

https://hub.docker.com/

### 命令

docker search xxx，不要盲目的相信这个命令了，还是使用上面的https://hub.docker.com/先查询一下比较好。

### centos7中文基础镜像

注意：centos8使用如下脚本报错。

如果你使用docker pull centos ,则下载的是centos8的版本。

#### 准备

下载对于的版本的centos，例如：7.8.2003

```
docker pull centos:7.8.2003
```

打标签为私有仓库

```shell
docker tag centos:7.8.2003 dockerdongyuit.cn:5000/centos:7.8.2003
```

存放到私有仓库

```
docker push  dockerdongyuit.cn:5000/centos:7.8.2003
```

#### 编辑Dockerfile

vi Dockerfile

```dockerfile
Dockerfile内容
# Docker file for date and locale set 
# VERSION 0.0.3
# Author: heige

#基础镜像
FROM dockerdongyuit.cn:5000/centos:7.8.2003

#作者
MAINTAINER Heige <909933699@qq.com>

# 设置YUM源
RUN rm -rf /etc/yum.repos.d/*.repo
RUN curl http://mirrors.163.com/.help/CentOS7-Base-163.repo > /etc/yum.repos.d/CentOS-Base.repo
RUN curl http://mirrors.aliyun.com/repo/epel-7.repo > /etc/yum.repos.d/epel-7.repo
RUN yum clean all
RUN yum makecache

#定义时区参数
ENV TZ=Asia/Shanghai

#设置时区
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo '$TZ' > /etc/timezone

#安装字符集
RUN yum -y install kde-l10n-Chinese glibc-common

#设置字符集
RUN localedef -c -f UTF-8 -i zh_CN zh_CN.utf8

#设置环境变量
ENV LC_ALL zh_CN.utf8
ENV LANG zh_CN.utf8
```

#### 构建和发布到私有仓库

docker build -t base/centos7zh:1.0.1 .

docker tag base/centos7zh:1.0.1 dockerdongyuit.cn:5000/centos7zh:1.0.1

docker push  dockerdongyuit.cn:5000/centos7zh:1.0.1

### centos7zh+jdk1.8镜像

#### 准备

Dockerfile目录下依赖于jdk目录，这个文件可以从通过tar -zxvf jdk-8u212-linux-x64.tar.gz解压获取到jdk目录。

vi Dockerfile

```dockerfile
# Docker file for date and locale set 
# VERSION 0.0.3
# Author: heige

#基础镜像
FROM dockerdongyuit.cn:5000/centos7zh:1.0.1

#作者
MAINTAINER Heige <909933699@qq.com>

# 拷贝jdk到容器(需要在Dockerfile相同目录下有jdk目录)
COPY jdk /usr/local/jdk
RUN ln -s /usr/local/jdk/bin/java /usr/bin/java
RUN ln -s /usr/local/jdk/bin/javac /usr/bin/javac
RUN ln -s /usr/local/jdk/jre /usr/local/jre

# 设置环境变量
ENV JAVA_HOME /usr/local/jdk
ENV JRE_HOME /usr/local/jre
ENV PATH $JAVA_HOME/bin:$PATH
ENV CLASSPATH .:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
```

docker build -t base/java:1.8 .

docker tag base/java:1.8 dockerdongyuit.cn:5000/java:1.8

docker push  dockerdongyuit.cn:5000/java:1.8

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

下面的配置会依赖这个redis1的docker实例名，如果需要部署多个可以替换这个redis1字样为redis2、redis3...。

因为redis容器启动用户polkitd，因此要修改data和log目录的所有者。config目录的配置文件都是只读的，即使修改也是在docker用户下直接修改了，因此所有者设置为docker了，这样docker用户修改，polkitd用户也可以读取。

```
su - root
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' >> /etc/rc.local
mkdir -p /home/docker/redis1/data
mkdir -p /home/docker/redis1/log
mkdir -p /home/docker/redis1/config
chown polkitd:polkitd /home/docker/redis1/data/
chown polkitd:polkitd /home/docker/redis1/log/
chown docker:docker /home/docker/redis1/config/
```

**下载redis.conf配置文件**

```shell
curl  http://download.redis.io/redis-stable/redis.conf >> /home/docker/redis1/config/redis.conf
```

vi /home/docker/redis1/config/redis.conf

在官网redis.conf的修改，使用下面的配置

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
maxmemory-policy volatile-lru # 对"过期集合(exipre)"中的数据采取LRU(近期最少使用)算法   

requirepass xxxxxx   #秘钥

logfile /log/redis.log # 指定log文件的位置
```

内存模式配置

```properties
# 保护模式设置为no
protected-mode no
# docker环境应该使用0.0.0.0绑定,安全有宿主的防火墙控制
bind 0.0.0.0
# 基于后台允许
daemonize no
# 允许的最大数据库数量
databases 100
# 禁止持久化
save ""
# 允许使用的最大内存空间
maxmemory 5368709120
# 对"过期集合(exipre)"中的数据采取LRU(近期最少使用)算法   
maxmemory-policy volatile-lru
#秘钥
requirepass 12345678
# 指定log文件的位置
logfile /log/redis.log
```

**运行redis**

```shell
docker run --name redis1 -p 6379:6379 \
-v /home/docker/redis1/config/redis.conf:/etc/redis/redis.conf \
-v /home/docker/redis1/log:/log \
--privileged --sysctl net.core.somaxconn=10240 \
-d redis redis-server /etc/redis/redis.conf
```

**查看redis日志**

```shell
tail -f /home/docker/redis1/log/redis.log
```

**telnet测试远程连接**

```
telnet localhost 6379
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
docker run -d --name rabbitmq1 -p 5672:5672 -p 15672:15672 -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=Rabbitmq-401 rabbitmq
```

运行界面管理插件

```shell
docker exec -it rabbitmq1 rabbitmq-plugins enable rabbitmq_management
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

### Nginx镜像

**下载镜像**

```
docker pull nginx
```

**创建nginx挂载目录**

```
mkdir -p /data/nginx/{conf,conf.d,html,logs}
```

**编辑nginx配置文件**

vi /data/nginx/conf/nginx.conf

```nginx
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    #include /etc/nginx/conf.d/*.conf; 注意：注释掉这个，否则你要仔细阅读这个目录下的default.conf文件内容
    # 添加内容
    
    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;
        #access_log  /var/log/nginx/host.access.log  main;

        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /usr/share/nginx/html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }
    
}
```

上面是官方的nginx.conf配置

如果需要配置server和upstream可以加入：

```nginx
    server {
        listen       80;
        server_name  当前服务器ip地址;

        charset utf-8;
    
		location /services {
            proxy_pass http://services;        
        }

    }

    upstream services {
        server 192.168.1.250:7070;
        keepalive 100;
    }


```

**验证nginx.conf**

第1个nginx为容器名

```shell
docker exec -it nginx nginx -t
```

**重启nginx**

nginx为容器名

```nginx
docker restart nginx
```

目前测试：docker exec -it nginx nginx -s reload不起作用。

**运行nginx**

```shell
docker run --name nginx -d --net host -v /data/nginx/conf/nginx.conf:/etc/nginx/nginx.conf  -v /data/nginx/logs:/var/log/nginx nginx
```

如果需要挂载html

```
-v /data/nginx/html:/usr/share/nginx/html
```

**开启防火墙**

```
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --reload
```

### jenkins镜像

**拉取jenkins镜像**

```shell
docker pull jenkins/jenkins:lts
```

这里拉取的是lts版本，这个版本是jenkins长期支持的版本。

**设置jenkins访问者权限**

```shell
mkdir -p /home/docker/jenkins/jenkins_home
chown -R 1000:1000 /home/docker/jenkins/jenkins_home
或者
启动jenkins指定为root用户，docker启动脚本加入 --user=root 
```

**启动jenkins**

```shell
docker run --name jenkins --env JAVA_OPTS="-Dhudson.model.DownloadService.noSignatureCheck=true -Duser.timezone=Asia/Shanghai" -p 10000:10000 -p 50000:50000 -v /home/docker/jenkins/jenkins_home:/var/jenkins_home -v /usr/local/jdk:/jdk -v /usr/local/maven:/maven -v /home/docker/maven_repo:/maven_repo -d jenkins/jenkins:lts
```

启动项说明：

--name 指定了docker的名称;

--env JAVA_OPTS=-Dhudson.model.DownloadService.noSignatureCheck=true 不进行插件的摘要验证；

--user=docker 使用docker用户启动jenkins。默认是jenkins用户启动，这需要为外界挂载目录设置拥有者1000；

-p 10000:10000映射http管理界面端口；

-p 50000:50000 映射jnpi端口；

-v /home/docker/jenkins/jenkins_home:/var/jenkins_home 挂载宿主机jenkins_home目录；

-v /usr/local/jdk:/jdk 宿主机的jdk目录，挂载到docker容器中；

-v /usr/local/maven:/maven 宿主机的maven目录，挂载到docker容器中；

-v /home/docker/maven_repo:/maven_repo 宿主机的maven_repo目录(maven本地仓库)，挂载到docker容器中；

-d jenkins/jenkins:lts 后台启动jenkins/jenkins:lts容器；

**修改插件源**

上面第一次启动后会在/data/jenkines_home(挂载宿主机)目录下生成很多文件，

修改/data/jenkines_home/hudson.model.UpdateCenter.xml文件，这里修改url为：

https://jenkins-zh.gitee.io/update-center-mirror/tsinghua/update-center.json，这是国内的插件镜像地址；

```xml
<?xml version='1.0' encoding='UTF-8'?>
<sites>
  <site>
    <id>default</id>
    <url>https://jenkins-zh.gitee.io/update-center-mirror/tsinghua/update-center.json</url>
  </site>
</sites>
```

再重新启动docker jenkines容器。http://ip:10000/restart

**http界面初始化jenkins系统**

http://ip:10000

第一次启动，初始化密码，这个密码可以在jenkins docker启动控制台输出上看到，把这个粘贴过来，点击确认；

```shell
Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

7dab5a023c5041c4b2437be2cec3cb72

```

第二步，插件初始化，这里可以先不用安装任何插件，选择第2个（自定义插件安装），并选择none（不安装插件）。

第三步，创建一个用户，正常创建就可以了。

以上jenkines的docker镜像安装和设置就完成了，jenkins的使用，可以见linux下的jenkins文档。

### 制作业务系统镜像

Dockerfile文件加入如下两个指令，在dockerfile的FROM指令后，ENTRYPOINT指令前：

```dockerfile
RUN useradd noroot -u 1000 -s /bin/bash
USER noroot
```

这里的-u指定了用户的sid，这里统一设置为1000，如果你的宿主机需要挂载目录到镜像，则你的宿主机挂载的目录需要，指定1000为拥有者。例如：chown -R 1000:1000 /data/jenkins_home

https://blog.csdn.net/yygydjkthh/article/details/47694929

可以在宿主机上通过ps -ef|grep xxx，xxx为进程名，例如：nginx、redis等，来观察是否是非root用户启动docker.

## docker swarm

### 防火墙设置

集群节点之间保证TCP 2377、TCP/UDP 7946和UDP 4789端口通信;

TCP端口2377集群管理端口;

TCP与UDP端口7946节点之间通讯端口;

TCP与UDP端口4789 overlay网络通讯端口;

```shell
firewall-cmd --zone=public --add-port=2377/tcp --permanent
firewall-cmd --zone=public --add-port=7946/tcp --permanent
firewall-cmd --zone=public --add-port=7946/udp --permanent
firewall-cmd --zone=public --add-port=4789/tcp --permanent
firewall-cmd --zone=public --add-port=4789/udp --permanent
firewall-cmd --reload
```

查看没有启动swarm状态

```
docker info |grep Swarm
```

Swarm: inactive

### 初始化管理节点

--advertise-addr 指定了对外提供集群管理的ip地址。

```
docker swarm init --advertise-addr 192.168.5.75
```

**查看已经启动了swarm状态**

```
docker info |grep Swarm
```

Swarm: active

**查看节点的状态**

docker node ls

```
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
y2yx1q4xl13d0qy3z8r7jkejj *   docker1             Ready               Active              Leader              19.03.8
```

### 加入管理节点

在上面**初始化管理节点**的服务器执行如下命令，输出的内容就要加入要执行的命令，在要加入的服务器节点上执行。

docker swarm join-token manager

```
docker swarm join --token SWMTKN-1-0iao02t4cq9mn709031gzg6p2e6v582mhcoorofzc5p27a0avx-cxmy21fwmyh80ykwwu0svxiys 192.168.5.75:2377
```

### 加入工作节点

在上面**初始化管理节点**的服务器执行如下命令，输出的内容就要加入要执行的命令，在要加入的服务器节点上执行。

docker swarm join-token worker

```
docker swarm join --token SWMTKN-1-0iao02t4cq9mn709031gzg6p2e6v582mhcoorofzc5p27a0avx-d256tdlucfbc3zs0tb3boralv 192.168.5.75:2377
```

### 查看节点状态

docker node ls

MANAGER STATUS状态说明：

Leader表示当前的管理者；

Reachable表示后补管理者；

空状态表示为工作节点；

```
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
mqn2hfdkanon8gknyhujaq64g *   docker1             Ready               Active              Leader              19.03.8
8lpn6i5wjsgprizf0ovll2xhh     docker2             Ready               Active              Reachable           19.03.8
ua3ihfmaq0izhjwfjwbxk5rlg     docker3             Ready               Active              Reachable           19.03.8

```

**加入管理节点的服务器上都会开启2377端口用户集群管理**

docker1是执行netstat -ant |grep 2377

```
tcp        0      0 192.168.5.75:37414      192.168.5.76:2377       ESTABLISHED
tcp        0      0 192.168.5.75:14936      192.168.5.77:2377       ESTABLISHED
```

docker2是执行netstat -ant |grep 2377

```
tcp        0      0 192.168.5.76:36970      192.168.5.75:2377       ESTABLISHED
tcp        0      0 192.168.5.76:36972      192.168.5.75:2377       ESTABLISHED
tcp        0      0 192.168.5.76:55988      192.168.5.77:2377       ESTABLISHED
```

docker3是执行netstat -ant |grep 2377

```
tcp        0      0 192.168.5.77:28908      192.168.5.75:2377       ESTABLISHED
tcp        0      0 192.168.5.77:5468       192.168.5.76:2377       ESTABLISHED
tcp        0      0 192.168.5.77:28912      192.168.5.75:2377       ESTABLISHED

```

### pull配置代理

```
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo vi /etc/systemd/system/docker.service.d/http-proxy.conf
设置代理服务器
[Service]
Environment="HTTP_PROXY=http://proxy_user:proxy_pass@proxy_server_ip:proxy_server_port"
重启docker
sudo systemctl daemon-reload
sudo systemctl restart docker
查看配置
systemctl show --property=Environment docker
验证
docker search nginx

```

### 创建网络

docker network create --subnet=192.168.5.0/24 -d overlay net168-5

查看

docker network ls

多出一个

```
fpfbg9hho9bw        net168-5            overlay             swarm
```

### 三节点部署nginx例子

docker service create --name nginx --publish 80:80 --network net168-5 --replicas 3 nginx

```
overall progress: 3 out of 3 tasks 
1/3: running   [==================================================>] 
2/3: running   [==================================================>] 
3/3: running   [==================================================>] 
verify: Service converged 

```

验证，查看三个节点的nginx

```
docker ps
```

http请求访问这三个节点

### 服务管理

#### 创建服务

```
create service create xxxx
```

##### 绑定宿主目录到docker容器目录

```
--mount type=bind,src=宿主目录,dst=容器目录
```

#### 发布配置

例如：下面把nginx.conf文件创建为一个**配置**

```
docker config create nginx-conf ./nginx/nginx.conf
```

这里会在集群中发布3个nginx，每个都使用上面创建的配置。

```
docker service create --name nginx --publish 80:8080 --network net168-1 --config source=nginx-conf --replicas 3 nginx
```

#### 发布标记

基于特定标记的发布，例如：把指定的镜像发布到拥有某个标记的机器上，例如：只发布到生产环境的机器上。

例如：先创建生产环境标记，标记是以key和value的格式出现的。如下env=proc为添加的标记，docker1集群上的某个nodeId。

```
docker node update --label-add env=proc docker1
```

发布nginx到生产环境，这里主要：--constraint node.labels.env==proc，指定了标记条件env==proc

```
docker service create --name nginx --publish 80:8080 --network net168-1 --config source=nginx-conf --replicas 3 --constraint node.labels.env==proc nginx
```



#### 停止服务

```
docker service rm xxxxx
```



### 查看服务日志

```
docker service logs xxxx 
```











docker service create --name nginx --publish 80:80 --network net168-5 --replicas 3 --mount  type=bind,src=/home/nginx/nginx/nginx.conf,dst=/etc/nginx/nginx.conf nginx





docker service create --name=mynginx --mount type=volume,source=/etc/nginx/conf,target=/usr/local/nginx/conf --publish-add published=80,target=80 docker.io/nginx



```
docker service create --name nginx --publish 80:80 --network net168-5 --replicas 3 --mount type=bind,src=/home/docker/nginx.conf,dst=/etc/nginx/nginx.conf  --mount type=bind,src=/home/docker/nginx_log,dst=/var/log/nginx  nginx
```



### 命令

#### 打标记

```
docker node update --label-add [标签] [节点名]
docker node update --label-rm [标签] [节点名]
docker node inspect -f {{".Spec.Labels"}} [节点名]
```

##### 加标记

例如：为节点docker1，加一个env-proc=1(KV结构)的标记

```
docker node update --label-rm env-proc=1 docker1
```

##### 删标记

例如：去掉节点docker1上的env=proc为key的标记

```
docker node update --label-rm env-proc docker1
```

##### 查看节点上的标记

```
docker node inspect -f {{".Spec.Labels"}} docker1 
```

## 安全

### 只允许用户访问一个特定的容器

```
更强的安全性
如果你打算只允许用户访问一个特定的容器，你可以写一个简单的脚本:

cat /usr/bin/docker-fedora
#!/bin/sh
docker run -ti --rm fedora /bin/sh
写好脚本之后，配置sudoers：

grep dwalsh /etc/sudoers
dwalsh        ALL=(ALL)       NOPASSWD: /usr/bin/docker-fedora
这个用户将仅能在没有权限限制下运行Fedora容器。
```



### FAQ

在3节点manager的情况下，如果两个节点down，则docker node ls命令：

Error response from daemon: rpc error: code = Unknown desc = The swarm does not have a leader. It's possible that too few managers are online. Make sure more than half of the managers are online.

https://blog.51cto.com/ligeo5210/2304294

https://zhuanlan.zhihu.com/p/25738959

https://www.cnblogs.com/fundebug/p/6823897.html

https://blog.csdn.net/luanpeng825485697/article/details/102745888