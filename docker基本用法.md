## docker基本用法

### 1.镜像相关操作

- 查找镜像

```bash
docker search centos
```

<pre><code>
docker search centos
</code></pre>

![image-20250325213549028](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250325213549028.png)

**解释说明：**

NAME: 镜像仓库源的名称

DESCRIPTION: 镜像的描述

OFFICIAL: 是否docker 官方发布

stars:类似Github 里面的star，表示点赞、喜欢的意思。

AUTOMATED: 自动构建。

- 下载镜像

```bash
docker pull centos:7
```

- 查看本地镜像

```bash
docker images
```

- 把镜像做成离线压缩包

```bash
docker save -o centos.tar.gz centos
```

- 解压离线镜像包

```bash
docker load -i centos.tar.gz 
```

- 删除镜像

```bash
docker rmi -f centos:latest
```

### 2.容器相关操作

- 以交互式方式启动并进入容器

```bash
docker run --name=hello -it centos /bin/bash
```

输入exit，退出容器，退出之后容器也会停止，不会再前台运行。

**docker run**运行并创建容器

**--name**容器的名字

**-i**交互式

**-t**分配伪终端

**centos:** 启动docker需要的镜像

**/bin/bash**说明你的shell类型为bash，bash shell是最常用的一种shell, 是大多数Linux发行版默认的shell。此外还有C shell等其它shell。

- 以守护进程方式启动容器

```bash
docker run --name=hello1 -td centos 

docker ps | grep hello1
```

**-d**在后台运行docker

- 查看正在运行的容器

```bash
# 查看正在运行的容器
docker ps

# 查看所有容器，包括运行和退出的容器
docker ps -a
```

- 停止容器

```bash
docker stop hello1
```

- 启动已经停止的容器

```bash
docker start hello1
```

- 进入容器

```bash
docker exec -it hello1 /bin/bash
```

- 删除容器

```bash
docker rm -f hello1
```

- 查看docker 帮助命令

```bash
docker --help 
```

## 通过docker部署nginx服务

```bash
docker run --name nginx -p  80 -itd centos:7
```

**-p** 把容器端口随机在物理机随机映射一个端口

```bash
docker ps | grep nginx

# 进入容器
docker exec -it nginx /bin/bash
```

![image-20250325233733662](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250325233733662.png)

进入容器后：

```bash
# yum换源
rm -rf /etc/yum.repos.d/*

curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo

yum clean all && yum makecache

yum install wget -y

# 容器内安装网络工具
yum install -y net-tools

# 安装 EPEL 仓库
yum install -y epel-release

# 更新缓存
yum makecache

# yum安装nginx
yum install nginx -y

# 安装文本编辑器vim
yum install vim-enhanced -y

# 查看容器的ip
ifconfig
```

创建静态页面：

```bash
mkdir /var/www/html -p 

cd /var/www/html/

vim index.html
```

粘贴内容：

```html
<html>
    <head>
        <title>nginx in docker</title>
    </head>
    <body>
        <h1>
            hello,My Name is master
        </h1>
    </body>
</html>
```

修改nginx配置文件中的root路径：

```bash
vim /etc/nginx/nginx.conf
```

修改该文件下server模块中的root路径为：

`root    /var/www/html`

在容器中启动nginx：

`/usr/sbin/nginx`

退出容器并查看：

```bash
docker ps

# 能查看到nginx容器在物理机映射的端口是32768
CONTAINER ID   IMAGE      COMMAND       CREATED          STATUS          PORTS                                     NAMES
018d31d655ef   centos:7   "/bin/bash"   47 minutes ago   Up 47 minutes   0.0.0.0:32768->80/tcp, :::32768->80/tcp   nginx
```

在浏览器访问：

http://192.168.179.139:32768

可以看到：

![image-20250326001619362](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250326001619362.png)

也可以查看容器的ip：

```bash
docker exec -it nginx /bin/bash

cat /etc/hosts

curl 172.17.0.2
```

![image-20250326001851684](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250326001851684.png)

**流量走向：**

访问物理节点ip:port（容器在物理节点映射的端口）--→容器ip:port（容器里部署的服务的端口）->

就可以访问到容器里部署的应用了

## dockerfile 语法详解

**Dockerfile** 是一个用来构建镜像的文本文件，文本内容包含了一条条构建镜像所需的指令和说明。

基于**Dockerfile**构建镜像可以使用`docker build`命令。`docker build`命令中使用 **-f**可以指定具体的dockerfile文件。

在nginx文件夹下创建**dockerfile**文件:

`mkdir nginx`

`cd nginx`

`vim dockerfile`

拉取文件：

`curl -o nginx/Centos-vault-8.5.2111.repo https://mirrors.aliyun.com/repo/Centos-vault-8.5.2111.repo`

编写dockerfile：

```dockerfile
FROM centos
MAINTAINER lly
RUN rm -rf /etc/yum.repos.d/*
COPY Centos-vault-8.5.2111.repo /etc/yum.repos.d/
RUN yum install wget -y
RUN yum install nginx -y
COPY index.html /usr/share/nginx/html/
EXPOSE 80
ENTRYPOINT ["/usr/sbin/nginx","-g","daemon off;"]
```

在nginx文件夹下创建index.html文件：`vim index.html`

```html
<html>
    <head>
        <title>page added to dockerfile</title>
    </head>
    <body>
        <h1>
            i am in df_test
        </h1>
    </body>
</html>
```

根据当前的dockerfile构建镜像：

```bash
docker build -t nginx:v1 .
```

**dockerfile构建过程：**

- **从基础镜像运行一个容器**

- **执行一条指令，对容器做出修改**

- **执行类似docker commit的操作，提交一个新的镜像层**

- **再基于刚提交的镜像运行一个新的容器**

- **执行dockerfile中的下一条指令，直至所有指令执行完毕**

### 1.FROM

基础镜像必须是可以下载下来的，定制的镜像都是基于FROM 的镜像，这里的centos就是定制需要的基础镜像。后续的操作都是基于centos镜像。

### 2.MAINTAINER

用于指定镜像的作者信息

### 3.RUN

RUN指定在当前镜像构建过程中要运行的命令。

包含两种模式：

- Shell模式

**`RUN <command>`** 

- exec模式

**`RUN ["executable"，"param1"，"param2"]`**

例如**`RUN ["/bin/bash", "-c", "echo hello"]`**

等价于**`/bin/bash -c echo hello`**

### 4.EXPOSE

仅仅只是声明端口。

**作用：**

1、帮助镜像使用者理解这个镜像服务的守护端口，以方便配置映射。

2、在运行时使用随机端口映射时，也就是`docker run -P`时，会自动随机映射EXPOSE 的端口。

3、可以是一个或者多个端口，也可以指定多个EXPOSE。

格式：EXPOSE <端口1> [<端口2>...]

### 5.CMD

类似于RUN指令，用于运行程序，但二者运行的时间点不同:

- CMD在`docker run`时运行。
- RUN 是在`docker build`构建镜像时运行的。

**作用**：为启动的容器指定默认要运行的程序，程序运行结束，容器也就结束。CMD 指令指定的程序可被

`docker run `命令行参数中指定要运行的程序所覆盖。

`CMD["executable"，"param1"，"param2"]`（exec模式）

`CMD command`（shell模式）

例子：

```dockerfile
# First dockerfile
FROM centos
MAINTAINER lly
RUN rm -rf /etc/yum.repos.d/*
COPY Centos-vault-8.5.2111.repo /etc/yum.repos.d/
RUN yum install wget -y
RUN yum install nginx -y
EXPOSE 80
CMD ["/usr/sbin/nginx", "-g", "daemon off;"]
```

构建镜像：

`docker build -t="dockerfile/test-cmd:v1" .`

基于上面构建的镜像运行一个容器：

`docker run -p 80 --name cmd_test2 -d dockerfile/test-cmd:v1`

容器将自动启动nginx服务，无需自行启动。

### 6.ENTERYPOINT

类似于CMD指令，但其不会被`docker run`的命令行参数指定的指令所覆盖，而且这些命令行参数会被当作参数送给**ENTRYPOINT**指令指定的程序。但是如果运行`docker run` 时使用了`--entrypoint` 选项，将覆盖entrypoint指令指定的程序。

优点：在执行`docker run`的时候可以指定`ENTRYPOINT`运行所需的参数。

**注意：**如果dockerfile 中如果存在多个`ENTRYPOINT` 指令，仅最后一个生效。

**`ENTERYPOINT ["executable", "param1", "param2"]`**（exec模式）

**`ENTERYPOINT command`** （shell模式）

可以搭配 CMD 命令使用：一般是变参才会使用 CMD ，这里的 CMD 等于是在给 ENTRYPOINT 传参，以下 

 示例会提到。

**示例：** 

假设已通过 Dockerfile 构建了 nginx:test 镜像：

```dockerfile
FROM nginx

ENTRYPOINT ["nginx", "-c"] # 定参

CMD ["/etc/nginx/nginx.conf"] # 变参
```

- **不传参运行**

`docker run nginx:test`

 容器内会默认运行以下命令，启动主进程。

`nginx -c /etc/nginx/nginx.conf ` 

-  **传参运行**

`docker run nginx:test -c /etc/nginx/new.conf`

容器内会默认运行以下命令，启动主进程(/etc/nginx/new.conf:假设容器内已有此文件)

`nginx -c /etc/nginx/new.conf`

###  7.COPY

 **`COPY<src>...<dest>`** 

**`COPY["<src>" ... "<dest>"]`** 

复制指令，从上下文目录中复制文件或者目录到容器里指定路径。*

格式：

**`COPY [--chown=<user>:<group>] <源路径 1>... <目标路径>`** 

**`COPY [--chown=<user>:<group>] ["<源路径 1>",... "<目标路径>"]`** 

**`[--chown=<user>:<group>]`**：可选参数，用户改变复制到容器内文件的拥有者和属组。

**<源路径>：源文件或者源目录，这里可以是通配符表达式，其通配符规则要满足 Go 的 filepath.Match** 

**规则。例如：** 

**`COPY hom* /mydir/`** 

**`COPY hom?.txt /mydir/`** 

**<目标路径>：容器内的指定路径，该路径不用事先建好，路径不存在的话，会自动创建。**

### 8.ADD

**`ADD <src>...<dest>`** 

**`ADD ["<src>"..."<dest>"]`**

ADD 指令和 COPY 的使用格式一致（同样需求下，官方推荐使用 COPY）。功能也类似，不同之处如下：

ADD 的优点：在执行 <源文件> 为 tar 压缩文件的话，压缩格式为 gzip, bzip2 以及 xz 的情况下，会自动复制并解压到 <目标路径>。

ADD 的缺点：在不解压的前提下，无法复制 tar 压缩文件。会令镜像构建缓存失效，从而可能会令镜像构建变得比较缓慢。具体是否使用，可以根据是否需要自动解压来决定。

**ADD vs COPY** 

**ADD 包含类似 tar 的解压功能** 

**如果单纯复制文件，dockerfile 推荐使用 COPY**

### 9.VOLUME

定义匿名数据卷。在启动容器时忘记挂载数据卷，会自动挂载到匿名卷。

作用：

1、避免重要的数据，因容器重启而丢失，这是非常致命的。

2、避免容器不断变大

格式：

**`VOLUME ["<路径 1>", "<路径 2>"...]`** 

**`VOLUME <路径>`** 

在启动容器 docker run 的时候，我们可以通过 -v 参数修改挂载点。 

**`VOLUME["/data"]`**

### 10.WORKDIR

指定工作目录。用 WORKDIR 指定的工作目录，会在构建镜像的每一层中都存在。（WORKDIR 指定的工作目录，必须是提前创建好的）。

docker build 构建镜像过程中的，每一个 RUN 命令都是新建的一层。只有通过 WORKDIR 创建的目录才会一直存在。

格式：

**`WORKDIR <工作目录路径>`** 

**`WORKDIR /path/to/workdir`** （填写绝对路径）

### 11.ENV

**设置环境变量** 

**`ENV <key> <value>`** 

**`ENV <key>=<value>...`** 

**以下示例设置 NODE_VERSION =6.6.6， 在后续的指令中可以通过 $NODE_VERSION 引用：** 

```dockerfile
ENV NODE_VERSION 6.6.6

RUN curl -SLO "https://nodejs.org/dist/v$NODE_VERSION/node-v$NODE_VERSION-linux-x64.tar.xz"
\
&& curl -SLO "https://nodejs.org/dist/v$NODE_VERSION/SHASUMS256.txt.asc
```

### 12.USER

用于指定执行后续命令的用户和用户组，这边只是切换后续命令执行的用户（用户和用户组必须提前已

经存在）。

### 13.ONBUILD

用于延迟构建命令的执行。简单的说，就是 Dockerfile 里用 ONBUILD 指定的命令，在本次构建镜

像的过程中不会执行（假设镜像为 test-build）。当有新的 Dockerfile 使用了之前构建的镜像 FROM

test-build ，这时执行新镜像的 Dockerfile 构建时候，会执行 test-build 的 Dockerfile 里的

ONBUILD 指定的命令。

格式：

**`ONBUILD <其它指令>`**

为镜像添加触发器

当一个镜像被其他镜像作为基础镜像时需要写上 OBNBUILD

这样会在构建时插入触发器指令。

### 14.LABEL

LABEL 指令用来给镜像添加一些元数据（metadata），以键值对的形式，语法格式如下：

LABEL <key>=<value> <key>=<value> <key>=<value> ...

比如我们可以添加镜像的作者：

LABEL org.opencontainers.image.authors="lly"

### 15.HEALTHCHECK

用于指定某个程序或者指令来监控 docker 容器服务的运行状态。

格式：

`HEALTHCHECK [选项] CMD <命令>`：设置检查容器健康状况的命令

`HEALTHCHECK NONE`：如果基础镜像有健康检查指令，使用这行可以屏蔽掉其健康检查指令

`HEALTHCHECK [选项] CMD <命令> `: 这边 CMD 后面跟随的命令使用，可以参考 CMD 的用法。

### 16.ARG

构建参数，与 ENV 作用一致。不过作用域不一样。ARG 设置的环境变量仅对 Dockerfile 内有效，也就

是说只有 `docker build `的过程中有效，构建好的镜像内不存在此环境变量。

构建命令` docker build` 中可以用` --build-arg <参数名>=<值>` 来覆盖。

格式：

`ARG <参数名>[=<默认值>]`

## dockerfile构建企业级Nginx镜像

`mkdir dockerfile`

`cd dockerfile`

`vim dockerfile`

```dockerfile
FROM centos
MAINTAINER lly
RUN rm -rf /etc/yum.repos.d/*
COPY Centos-vault-8.5.2111.repo /etc/yum.repos.d/
RUN yum install wget -y
RUN yum install nginx -y
COPY index.html /usr/share/nginx/html/
EXPOSE 80
ENTRYPOINT ["/usr/sbin/nginx","-g","daemon off;"]
```

`vim index.html`

```html
<html> 
	<head> 
 		<title>page added to dockerfile</title> 
	</head> 
	<body> 
 		<h1> Hello World! </h1> 
	</body>
</html>
```

构建镜像：**`docker build -t="lly/nginx:v1" .`**

基于刚才的镜像启动容器：

**`docker run -d -p 80 --name html2 lly/nginx:v1`**

## dockerfile构建tomcat镜像

`mkdir tomcat8`

`cd tomcat8`

**把 apache-tomcat-8.0.26.tar.gz 和 jdk-8u45-linux-x64.rpm 传到这个目录下**

编写dockerfile：

`vim dockerfile`

```dockerfile
FROM centos 
MAINTAINER lly 
RUN rm -rf /etc/yum.repos.d/* 
COPY Centos-vault-8.5.2111.repo /etc/yum.repos.d/
RUN yum install wget -y 
ADD jdk-8u45-linux-x64.rpm /usr/local/ 
ADD apache-tomcat-8.0.26.tar.gz /usr/local/ 
RUN cd /usr/local && rpm -ivh jdk-8u45-linux-x64.rpm 
RUN mv /usr/local/apache-tomcat-8.0.26 /usr/local/tomcat8 
EXPOSE 8080
```

构建镜像：

`docker build -t="tomcat8:v1" .`

运行一个容器：

`docker run --name tomcat8 -itd -p 8080 tomcat8:v1`

进入到容器:

`docker exec -it tomcat8 /bin/bash`

启动 tomcat：

`/usr/local/tomcat8/bin/startup.sh`

查看进程，看看是否启动成功

`ps -ef | grep tomcat`

**刚才我们在构建 tomcat 镜像时候，基于镜像运行容器，但是需要进入到容器，手动启动 tomcat 服务，**

**那如果想要启动容器，tomcat 也自动起来，需要按照如下方法构建镜像：**

```dockerfile
FROM centos 
MAINTAINER xianchao 
RUN rm -rf /etc/yum.repos.d/* 
COPY Centos-vault-8.5.2111.repo /etc/yum.repos.d/ 
RUN yum install wget -y 
ADD jdk-8u45-linux-x64.rpm /usr/local/ 
ADD apache-tomcat-8.0.26.tar.gz /usr/local/ 
RUN cd /usr/local && rpm -ivh jdk-8u45-linux-x64.rpm 
RUN mv /usr/local/apache-tomcat-8.0.26 /usr/local/tomcat8 
ENTRYPOINT /usr/local/tomcat8/bin/startup.sh && tail -F /usr/local/tomcat8/logs/catalina.out
```

