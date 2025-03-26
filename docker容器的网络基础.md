

## docker容器的网络基础

**docker run 创建 Docker 容器时，可以用--net 选项指定容器的网络模式，Docker 有以下 4 种网络模式：** 

- **bridge 模式：使--net =bridge 指定，默认设置；** 

- **host 模式：使--net =host 指定；** 

- **none 模式：使--net =none 指定；** 

- **container 模式：使用--net =container:NAME orID 指定。**

### 1.docker0:

安装docker的时候，会生成一个docker0的虚拟网桥

### 2.Linux虚拟网桥的特点:

可以设置ip地址，相当于拥有一个隐藏的虚拟网卡。

![image-20250325134323511](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250325134323511.png)

每运行一个docker容器都会生成一个veth设备对，这个veth一个接口在容器里，一个接口在物理机上。

![image-20250325134742187](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250325134742187.png)

### 3.安装网桥管理工具:

```bash
yum install bridge-utils -y
```

使用：

```bash
brctl show
```

可以查看到有一个docker0的网桥设备，下面有很多接口，每个接口都表示一个启动的docke容器。

![image-20250325142111035](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250325142111035.png)

### 4.docker容器的互联

`mkdir inter-image`



下面用到的镜像的dockerfile文件如下：

```dockerfile
FROM centos
MAINTAINER lly
RUN rm -rf /etc/yum.repos.d/*
COPY Centos-vault-8.5.2111.repo /etc/yum.repos.d/
RUN yum install wget -y
RUN yum install nginx -y
EXPOSE 80
CMD /bin/bash
```

使用命令构建镜像

```bash
docker build -t="inter-image" .
```

允许所有容器间互联（也就是访问）

**第一种方法：**

例： 

（1）基于上面的 inter-image 镜像启动第一个容器 test1

**`docker run --name test1 -itd inter-image`** 

**进入到容器里面启动 nginx：** 

**`/usr/sbin/nginx -t`**

（2）基于上面的 inter-image 镜像启动第二个容器 test2

**`docker run --name test2 -itd inter-image`** 

（3）进入到 test1 容器和 test2 容器，可以看两个容器的 ip，分别是

172.17.0.20 和 172.17.0.21

**`docker exec -it test2 /bin/bash`** 

`ping 172.17.0.20` 可以看见能 ping 同 test1 容器的 ip

**`curl http://172.17.0.20`**

可以访问到 test1容器的内容

上述方法假如 test1 容器重启，那么在启动就会重新分配 ip 地址，所以为了使 ip 地址变了也可以访问

可以采用下面的方法。

**第二种方法：** 

可以给容器起一个代号，这样可以直接以代号访问，避免了容器重启 ip 变化带来的问题

**`--link`** 

**`docker run --link=[CONTAINER_NAME]:[ALIAS] [IMAGE][COMMAND]`** 

例：

1.启动一个 test3 容器

**`docker run --name test3 -itd inter-image /bin/bash`** 

2.启动一个 test5 容器，--link 做链接，那么当我们重新启动 test3 容器时，就算 ip 变了，也没关系，

我们可以在 test5 上 ping 别名 webtest

**`docker run --name test5 -itd --link=test3:webtest inter-image /bin/bash`** 

3.test3 和 test5 的 ip 分别是 172.17.0.22 和 172.17.0.24

4.重启 test3 容器

**`docker restart test3`** 

发现 ip 变成了 172.17.0.25

5.进入到 test5 容器

**`docker exec -it test5 /bin/bash`** 

ping test3 容器的 ip 别名 webtest 可以 ping 通，尽管 test3 容器的 ip 变了也可以通

### 5.docker 容器的网络模式

#### none 模式

Docker 网络 none 模式是指创建的容器没有网络地址，只有 lo 网卡

`docker run -itd --name none --net=none --privileged=true centos `

**` docker exec -it none /bin/bash`** 

**`ip addr`**

只有本地 lo 地址

#### container模式

Docker 网络 container 模式是指，创建新容器的时候，通过--net container 参数，指定其和已经存

在的某个容器共享一个 Network Namespace。如下图所示，右方黄色新创建的 container，其网卡共享左

边容器。因此就不会拥有自己独立的 IP，而是共享左边容器的 IP 172.17.0.2,端口范围等网络资源，两

个容器的进程通过 lo 网卡设备通信。

![image-20250326214238221](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250326214238221.png)

`docker run --name container2 --net=container:none -it --privileged=true centos`

![image-20250326214754795](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250326214754795.png)

#### bridge 模式

默认选择 bridge 的情况下，容器启动后会通过 DHCP 获取一个地址。

创建桥接网络

`docker run --name bridge -it --privileged=true centos bash`

![image-20250326214854632](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250326214854632.png)

#### host 模式

Docker 网络 host 模式是指共享宿主机的网络

 `docker run --name host -it --net=host --privileged=true centos bash`
