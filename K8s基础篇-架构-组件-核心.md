## Kubernetes简介

**kubernetes** 是可移植、可扩展、开源的容器管理平台，是谷歌 Borg 的开源版本，简称 k8s ，它可以创建应用、更新应用、回滚应用，也可实现应用的扩容缩容，做到故障自恢复。

- 可移植：基于镜像可从一个环境迁移到另一个环境
- 可扩展： k8s 集群可以横向扩展、根据流量实现自动扩缩容
- 开源的：源代码已经公开了，可以被用户免费使用，可以二次开发

**可以对容器自动化部署、自动化扩缩容、跨主机管理等；**
**可以对代码进行灰度发布、金丝雀发布、蓝绿发布、滚动更新等；**
**具有完整的监控系统和日志收集平台，具有故障自恢复的能力。**

### Kubernetes 架构

k8s 的物理架构是 master/node 模式：
K8S 集群至少需要一个主节点 (Master) 和多个工作节点 (Worker) ， Master 节点是集群的控制节点，负责整个集群的
管理和控制，主要用于暴露 API 、调度部署和对节点进行管理。工作节点主要是运行容器的。
单 master 节点架构图如下：

![image-20250327193416260](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250327193416260.png)

### kubernetes 组件

![image-20250327194225307](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250327194225307.png)

#### Master node
- **apiserver**
- **scheduler**
- **controller-manager**
- Etcd
- calico
- docker

#### Work node
- **kubelet**
- **kube-proxy**
- **Calico**
- Coredns
- docker

**kubectl：** 管理 k8s 的命令行工具，可以操作 k8s 中的资源对象，如增删改查等。
**etcd：** 是一个高可用的键值数据库，存储 k8s 的资源状态信息和网络信息的， etcd 中的数据变更是通过 api server进行的。
**apiserver：** 提供 k8s api ，是整个系统的对外接口，提供资源操作的唯一入口，供客户端和其它组件调用，提供了k8s 各类资源对象（ pod,deployment,Service 等）的增删改查，是整个系统的数据总线和数据中心，并提供认证、授权、访问控制、 API 注册和发现等机制，并将操作对象持久化到 etcd 中。
**scheduler：**负责 k8s 集群中 pod 的调度的 ， scheduler 通过与 apiserver 交互监听到创建 Pod 副本的信息后，它会检索所有符合该 Pod 要求的工作节点列表，开始执行 Pod 调度逻辑。调度成功后将 Pod 绑定到目标节点上，相当于“调度室”。
**Controller-Manager：**作为集群内部的管理控制中心，负责集群内的 Node 、 Pod 副本、服务端点（ Endpoint ）、命名空间（ Namespace ）、服务账号（ ServiceAccount ）、资源定额（ResourceQuota ）的管理，当某个 Node 意外宕机时， Controller Manager 会及时发现并执行自动化修复流程，确保集群始终处于预期的工作状态。

每个 Controller 通过 API Server 提供的接口实时监控整个集群的每个资源对象的当前状态，当发生各种故障导致系统状态发生变化时，会尝试将系统状态修复到“期望状态”。

kubelet ： 每个 Node 节点上的 kubelet 定期就会调用 API Server 的 REST 接口报告自身状态， API Server 接收这些信
息后，将节点状态信息更新到 etcd 中。 kubelet 也通过 API Server 监听 Pod 信息，从而对 Node 机器上的 POD 进行管理
，如创建、删除、更新 Pod
kube-proxy ：提供网络代理和负载均衡，是实现 service 的通信与负载均衡机制的重要组件， kube-proxy 负责为 Pod
创建代理服务，从 apiserver 获取所有 service 信息，并根据 service 信息创建代理服务，实现 service 到 Pod 的请求
路由和转发，从而实现 K8s 层级的虚拟转发网络，将到 service 的请求转发到后端的 pod 上。
Cordns ： CoreDNS 其实就是一个 DNS 服务，而 DNS 作为一种常见的服务发现手段，很多开源项目以及工程师都会
使用 CoreDNS 为集群提供服务发现的功能， Kubernetes 就在集群中使用 CoreDNS 解决服务发现的问题。

Calico ： 是一套开源的网络和网络安全方案，用于容器、虚拟机、宿主机之前的网络连接，可以用在
kubernetes 、 OpenShift 、 DockerEE 、 OpenStrack 等 PaaS 或 IaaS 平台上。
Docker ：容器运行时，负责启动容器的，在 k8s1.20 版本之后建议废弃 docker ，使用 container 作为容器运行时

### kubernetes 核心资源

#### 1.Pod

Pod 是 Kubernetes 中的最小调度单元， k8s 是通过定义一个 Pod 的资源，然后在 Pod 里面运行容器，容器需要指
定镜像，用来运行具体的服务。 Pod 代表集群上正在运行的一个进程，一个 Pod 封装一个容器（也可以封装多个容
器）， Pod 里的容器共享存储、网络等。也就是说，应该把整个 pod 看作虚拟机，然后每个容器相当于运行在虚拟
机的进程。

在 K8s 中，所有的资源都可以使用一个 yaml 配置文件来创建，创建 Pod 也可以使用 yaml 配置文件。

#### 2.label

label 是标签的意思， k8s 中的资源对象大都可以打上标签，如 Node 、 Pod 、 Service 等，一个资源可以绑定任意多个 label ， k8s 通过 Label 可实现多维度的资源分组管理，后续可通过 Label Selector 查询和筛选拥有某些 Label 的资源对象，例如创建一个 Pod ，给定一个 Label 是 app=tomcat ，那么 service 可以通过 label selector 选择拥有 app=tomcat 的 pod ，和其相关联，也可通过 app=tomcat 删除拥有该标签的 Pod 资源。

![image-20250327234825431](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250327234825431.png)

#### 3.Deployment

Replicaset 是 Kubernetes 中的副本控制器，管理 Pod ，使 pod 副本的数量始终维持在预设的个数。
Deployment 是管理 Replicaset 和 Pod 的副本控制器， Deployment 可以管理多个 Replicaset ，是比 Replicaset更高级的控制器，也即是说在创建 Deployment 的时候，会自动创建 Replicaset ，由 Replicaset 再创建Pod ， Deployment 能对 Pod 扩容、缩容、滚动更新和回滚、维持 Pod 数量。

![image-20250327235018178](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250327235018178.png)

#### 4.Service

在 kubernetes 中， Pod 是有生命周期的，如果 Pod 重启 IP 很有可能会发生变化。如果我们的服务都是将Pod 的 IP 地址写死， Pod 的挂掉或者重启，和刚才重启的 pod 相关联的其他服务将会找不到它所关联的Pod ，为了解决这个问题，在 kubernetes 中定义了 service 资源对象， Service 定义了一个服务访问的入口，客户端通过这个入口即可访问服务背后的应用集群实例， service 是一组 Pod 的逻辑集合，这一组Pod 能够被 Service 访问到，通常是通过 Label Selector实现的。

### Kubeadm安装高可用的k8s集群

**k8s 环境规划：**

**podSubnet（pod 网段） 10.244.0.0/16** 

**serviceSubnet（service 网段）: 10.10.0.0/16** 

**实验环境规划：**

**操作系统：centos7.6**

**配置： 4Gib 内存/6vCPU/100G 硬盘**

#### **（1）初始化安装 k8s 集群的实验环境**

##### 1.修改机器ip，变成静态ip

`vim /etc/sysconfig/network-scripts/ifcfg-ens33`

```bash
TYPE=Ethernet 
PROXY_METHOD=none 
BROWSER_ONLY=no 
BOOTPROTO=static 
IPADDR=192.168.179.140 #ip 地址，需要跟自己电脑所在网段一致
NETMASK=255.255.255.0 #子网掩码，需要跟自己电脑所在网段一致
GATEWAY=192.168.179.2 #网关，在自己电脑打开 cmd，输入 ipconfig /all 可看到
DNS1=192.168.179.2 #DNS，在自己电脑打开 cmd，输入 ipconfig /all 可看到
DEFROUTE=yes 
IPV4_FAILURE_FATAL=no 
IPV6INIT=yes 
IPV6_AUTOCONF=yes 
IPV6_DEFROUTE=yes 
IPV6_FAILURE_FATAL=no 
IPV6_ADDR_GEN_MODE=stable-privacy 
NAME=ens33  #网卡名字，跟 DEVICE 名字保持一致即可
DEVICE=ens33 #网卡设备名，大家 ip addr 可看到自己的这个网卡设备名，每个人的机器可能这个名
字不一样，需要写自己的
ONBOOT=yes #开机自启动网络，必须是 yes
```

重启网络服务：

**`service network restart`**

##### 2.配置主机名

**在 192.168.179.140 上执行如下：** 

**`hostnamectl set-hostname master1 && bash`**

**在 192.168.179.141 上执行如下：** 

**`hostnamectl set-hostname master2 && bash`**

**在 192.168.179.142 上执行如下：** 

**`hostnamectl set-hostname node1 && bash`**

##### **3.配置主机相互之间通过主机名互相访问**

**修改每台机器的/etc/hosts 文件，增加如下三行：** 

**`vim /etc/hosts`**

```
192.168.179.140 master1

192.168.179.141 master2 

192.168.179.142 node1
```

##### 4.**配置主机之间无密码登录**

在每台机器上执行：

```bash
ssh-keygen

# 把本地生成的密钥文件和私钥文件拷贝到远程主机

ssh-copy-id master1

ssh-copy-id master2

ssh-copy-id node1
```

##### 5.**关闭交换分区 swap，提升性能**

每台机器上都执行：

**临时关闭：**

```bash
swapoff -a
```

**永久关闭：注释 swap 挂载，给 swap 这行开头加一下注释**

**`vim /etc/fstab`**

**`#/dev/mapper/centos-swap swap swap defaults 0 0`**

**如果是克隆的虚拟机，需要删除 UUID**

**Swap 是交换分区，如果机器内存不够，会使用 swap 分区，但是 swap 分区的性能较低，k8s 设计的时候为了能提升性能，默认是不允许使用交换分区的。Kubeadm 初始化的时候会检测 swap 是否关闭，如果没关闭，那就初始化失败。如果不想要关闭交换分区，安装 k8s 的时候可以指定--ignore preflight-errors=Swap 来解决。**

##### 6.修改机器内核参数

每台机器上执行：

````bash
modprobe br_netfilter

echo "modprobe br_netfilter" >> /etc/profile

vim /etc/sysctl.d/k8s.conf
```bash
net.bridge.bridge-nf-call-ip6tables = 1 
net.bridge.bridge-nf-call-iptables = 1 
net.ipv4.ip_forward = 1
```

sysctl -p /etc/sysctl.d/k8s.conf

modprobe br_netfilter

lsmod | grep br_netfilter
````

**在运行时配置内核参数 -p 从指定的文件加载系统参数，如不指定即从/etc/sysctl.conf 中加载。**

**net.ipv4.ip_forward 是数据包转发：**

**出于安全考虑，Linux 系统默认是禁止数据包转发的。所谓转发即当主机拥有于一块的网卡时，其中一块收到数据包，根据数据包的目的 ip 地址将数据包发往本机另一块网卡，该网卡根据路由表继续发送数据包。这通常是路由器所要实现的功能。要让 Linux 系统具有路由转发功能，需要配置一个 Linux 的内核参数 net.ipv4.ip_forward。这个参数指定了 Linux 系统当前对路由转发功能的支持情况；其值为 0 时表示禁止进行 IP 转发；如果是 1,则说明 IP 转发功能已经打开。**

##### 7.关闭firewalld防火墙

```bash
[root@master1 ~]# systemctl stop firewalld && systemctl disable firewalld 
[root@master2 ~]# systemctl stop firewalld && systemctl disable firewalld 
[root@node1 ~]# systemctl stop firewalld && systemctl disable firewalld
```

**关闭selinux：**

```bash
# 修改 selinux 配置文件之后，重启机器，selinux 配置才能永久生效
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

# 显示 Disabled 说明 selinux 已经关闭
getenforce
```

##### 8.配置阿里云的repo源

**在三个机器上操作：**

```bash
# 安装 rzsz 命令
yum install lrzsz -y

# 安装 scp命令
yum install openssh-clients

# 备份基础 repo 源
mkdir /root/repo.bak

cd /etc/yum.repos.d/

mv * /root/repo.bak/

# 下载阿里云的 repo 源
把 CentOS-Base.repo 文件上传到主机的/etc/yum.repos.d/目录下

# 配置国内阿里云 docker 的 repo 源
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

##### 9.**配置安装 k8s 组件需要的阿里云的 repo 源**

````bash
vim /etc/yum.repos.d/kubernetes.repo

```bash
[kubernetes] 
name=Kubernetes 
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/ 
enabled=1 
gpgcheck=0
```

#将 master1 上 Kubernetes 的 repo 源复制给 master2 和 node1
scp /etc/yum.repos.d/kubernetes.repo master2:/etc/yum.repos.d/

scp /etc/yum.repos.d/kubernetes.repo node1:/etc/yum.repos.d/
````

##### 10.**配置时间同步**

````bash
#安装 ntpdate 命令
yum install ntpdate -y

ntpdate cn.pool.ntp.org

crontab -e
```bash
* */1 * * * /usr/sbin/ntpdate cn.pool.ntp.org
```

# 重启 crond 服务
service crond restart
````

##### 11.**开启 ipvs**

把 ipvs.modules 上传到 master1 机器的/etc/sysconfig/modules/目录下。

```bash
cd /etc/sysconfig/modules/

# master1
chmod 777 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep ip_vs

scp /etc/sysconfig/modules/ipvs.modules master2:/etc/sysconfig/modules/

scp /etc/sysconfig/modules/ipvs.modules node1:/etc/sysconfig/modules/

# master2
chmod 777 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep ip_vs

# node1
chmod 777 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep ip_vs
```

##### 12.**安装基础软件包**

```bash
yum install -y  wget net-tools nfs-utils lrzsz gcc gcc-c++ make cmake libxml2-devel openssl-devel curl curl-devel unzip sudo ntp libaio-devel wget vim ncurses-devel autoconf automake zlib-devel python-devel epel-release openssh-server socat ipvsadm conntrack 
```

##### 13.**安装 iptables**

```bash
# 安装 iptables
yum install iptables-services -y

# 禁用 iptables
service iptables stop && systemctl disable iptables

# 清空防火墙规则
iptables -F
```

#### （2）安装docker服务

在三台机器上安装：

##### 安装docker-ce

```bash
# 安装 docker-ce
yum install docker-ce-20.10.6 docker-ce-cli-20.10.6 containerd.io -y

systemctl start docker && systemctl enable docker && systemctl status docker
```

##### 配置docker镜像加速器和驱动

````bash
vim /etc/docker/daemon.json

```bash
{ 
 "registry-mirrors":["https://rsbud4vc.mirror.aliyuncs.com","https://registry.docker-cn.com","https://docker.mirrors.ustc.edu.cn","https://dockerhub.azk8s.cn","http://hub-mirror.c.163.com","http://qtid6917.mirror.aliyuncs.com", "https://rncxm540.mirror.aliyuncs.com"], "exec-opts": ["native.cgroupdriver=systemd"]
 }
```

systemctl daemon-reload && systemctl restart docker

systemctl status docker
````

#### （3）**安装初始化 k8s 需要的软件包**

在三台机器上完成安装：

```bash
yum install -y kubelet-1.20.6 kubeadm-1.20.6 kubectl-1.20.6

systemctl enable kubelet && systemctl start kubelet

systemctl status kubelet
```

**面可以看到 kubelet 状态不是 running 状态，这个是正常的，不用管，等 k8s 组件起来这个kubelet 就正常了。**

#### （4）**通过 keepalive+nginx 实现 k8s apiserver 节点高可用**

**配置 epel 源**

把 epel.repo 上传到 xianchaomaster1 的`/etc/yum.repos.d` 目录下，这样才能安装 keepalived和nginx。

```bash
#把 epel.repo 拷贝到远程主机 xianchaomaster2 和 xianchaonode1 上 
scp /etc/yum.repos.d/epel.repo master2:/etc/yum.repos.d/ 
scp /etc/yum.repos.d/epel.repo node1:/etc/yum.repos.d/
```

**1、安装 nginx 主备**

```bash
# 在 master1 和 master2 上做 nginx 主备安装 
yum install nginx keepalived nginx-mod-stream -y 
yum install nginx keepalived nginx-mod-stream -y
```

**2、修改 nginx 配置文件，主备一样**

````bash
vim /etc/nginx/nginx.conf

```bash
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

# 四层负载均衡，为两台Master apiserver组件提供负载均衡
stream {

    log_format  main  '$remote_addr $upstream_addr - [$time_local] $status $upstream_bytes_sent';

    access_log  /var/log/nginx/k8s-access.log  main;

    upstream k8s-apiserver {
       server 192.168.179.140:6443;   # Master1 APISERVER IP:PORT
       server 192.168.179.141:6443;   # Master2 APISERVER IP:PORT
    }
    
    server {
       listen 16443; # 由于nginx与master节点复用，这个监听端口不能是6443，否则会冲突
       proxy_pass k8s-apiserver;
    }
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    server {
        listen       80 default_server;
        server_name  _;

        location / {
        }
    }
}
```
````

**3、keepalive 配置**

主 keepalived

````bash
vim /etc/keepalived/keepalived.conf

```bash
global_defs { 
   notification_email { 
     acassen@firewall.loc 
     failover@firewall.loc 
     sysadmin@firewall.loc 
   } 
   notification_email_from Alexandre.Cassen@firewall.loc  
   smtp_server 127.0.0.1 
   smtp_connect_timeout 30 
   router_id NGINX_MASTER
} 

vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
}

vrrp_instance VI_1 { 
    state MASTER 
    interface ens33  # 修改为实际网卡名
    virtual_router_id 51 # VRRP 路由 ID实例，每个实例是唯一的 
    priority 100    # 优先级，备服务器设置 90 
    advert_int 1    # 指定VRRP 心跳包通告间隔时间，默认1秒 
    authentication { 
        auth_type PASS      
        auth_pass 1111 
    }  
    # 虚拟IP
    virtual_ipaddress { 
        192.168.179.199/24
    } 
    track_script {
        check_nginx
    } 
}
```

vim /etc/keepalived/check_nginx.sh
```bash
#!/bin/bash 
count=$(ps -ef |grep nginx | grep sbin | egrep -cv "grep|$$") 
if [ "$count" -eq 0 ];then 
 systemctl stop keepalived 
fi
```

chmod +x /etc/keepalived/check_nginx.sh
````

备 keepalive

````bash
vim /etc/keepalived/keepalived.conf

```bash
global_defs { 
   notification_email { 
     acassen@firewall.loc 
     failover@firewall.loc 
     sysadmin@firewall.loc 
   } 
   notification_email_from Alexandre.Cassen@firewall.loc  
   smtp_server 127.0.0.1 
   smtp_connect_timeout 30 
   router_id NGINX_MASTER
} 

vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
}

vrrp_instance VI_1 { 
    state BACKUP
    interface ens33  # 修改为实际网卡名
    virtual_router_id 51 # VRRP 路由 ID实例，每个实例是唯一的 
    priority 90    # 优先级，备服务器设置 90 
    advert_int 1    # 指定VRRP 心跳包通告间隔时间，默认1秒 
    authentication { 
        auth_type PASS      
        auth_pass 1111 
    }  
    # 虚拟IP
    virtual_ipaddress { 
        192.168.179.199/24
    } 
    track_script {
        check_nginx
    } 
}
```

vim /etc/keepalived/check_nginx.sh

```bash
#!/bin/bash 
count=$(ps -ef |grep nginx | grep sbin | egrep -cv "grep|$$") 
if [ "$count" -eq 0 ];then 
 systemctl stop keepalived 
fi
```

chmod +x /etc/keepalived/check_nginx.sh
````

**4.启动服务**

```bash
systemctl daemon-reload 

yum install nginx-mod-stream -y 

systemctl start nginx 

systemctl start keepalived 

systemctl enable nginx keepalived 

systemctl status keepalived
```

**5、测试 vip 是否绑定成功**

**`ip addr`**

```bash
[root@master1 ~]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:4d:35:f9 brd ff:ff:ff:ff:ff:ff
    inet 192.168.179.140/24 brd 192.168.179.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet 192.168.179.199/24 scope global secondary ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::95c6:452f:f0fb:5745/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

6.测试keepalived

```bash
# 停掉 master1 上的 nginx。Vip 会漂移到 master2 
service nginx stop 

ip addr

# 启动 master1 上的 nginx 和 keepalived，vip 又会漂移回来
systemctl daemon-reload 
systemctl start nginx 
systemctl start keepalived 
ip addr
```

#### （5）kubeadm初始化k8s集群

````bash
# 在 master1 上创建 kubeadm-config.yaml 文件： 
cd /root/ 
vim kubeadm-config.yaml 

```bash
apiVersion: kubeadm.k8s.io/v1beta2 
kind: ClusterConfiguration 
kubernetesVersion: v1.20.6 
controlPlaneEndpoint: 192.168.179.199:16443 
imageRepository: registry.aliyuncs.com/google_containers
apiServer: 
 certSANs: 
 - 192.168.179.140 
 - 192.168.179.141 
 - 192.168.179.142 
 - 192.168.179.199 
networking: 
 podSubnet: 10.244.0.0/16 
 serviceSubnet: 10.10.0.0/16 
--- 
apiVersion: kubeproxy.config.k8s.io/v1alpha1 
kind: KubeProxyConfiguration 
mode: ipvs
```
````

```bash
# 把初始化 k8s 集群需要的离线镜像包上传到 master1、master2、node1机器上，手动解压： 
[root@master1 ~] docker load -i k8simage-1-20-6.tar.gz 
[root@master2 ~] docker load -i k8simage-1-20-6.tar.gz 
[root@node1 ~] docker load -i k8simage-1-20-6.tar.gz 
[root@master1] kubeadm init --config kubeadm-config.yaml --ignore-preflight-errors=SystemVerification
```

注：`--image-repository registry.aliyuncs.com/google_containers`：手动指定仓库地址为
`registry.aliyuncs.com/google_containers`。kubeadm 默认从 k8s.grc.io 拉取镜像，但是 k8s.gcr.io
访问不到，所以需要指定从 `registry.aliyuncs.com/google_containers` 仓库拉取镜像。

**安装完成后保存：**

````bash

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:
```bash
  kubeadm join 192.168.179.199:16443 --token p601o3.sde3fcbwsec2v0ka \
    --discovery-token-ca-cert-hash sha256:6162c7fbbca43378af14677c88083d93999f7543555ba60e9587b915a06859f8 \
    --control-plane 
```
# 上面命令是把 master 节点加入集群，需要保存下来
Then you can join any number of worker nodes by running the following on each as root:
```bash
kubeadm join 192.168.179.199:16443 --token p601o3.sde3fcbwsec2v0ka \
    --discovery-token-ca-cert-hash sha256:6162c7fbbca43378af14677c88083d93999f7543555ba60e9587b915a06859f8 
```
# 上面命令是把 node 节点加入集群，需要保存下来
````

```bash
# 配置 kubectl 的配置文件 config，相当于对 kubectl 进行授权，这样 kubectl 命令可以使用这个证书对 k8s 集群进行管理 
[root@master1 ~] mkdir -p $HOME/.kube 
[root@master1 ~] sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config 
[root@master1 ~] sudo chown $(id -u):$(id -g) $HOME/.kube/config 
[root@master1 ~] kubectl get nodes
```

#### （6）**扩容 k8s 集群-添加 master 节点**

```bash
#把 master1 节点的证书拷贝到 master2 上 
# 在 master2 创建证书存放目录： 
[root@master2 ~] cd /root && mkdir -p /etc/kubernetes/pki/etcd && mkdir -p ~/.kube/ 
#把 master1 节点的证书拷贝到 master2 上： 
[root@master1 ~] scp /etc/kubernetes/pki/ca.crt master2:/etc/kubernetes/pki/ 
[root@master1 ~] scp /etc/kubernetes/pki/ca.key master2:/etc/kubernetes/pki/ 
[root@master1 ~] scp /etc/kubernetes/pki/sa.key master2:/etc/kubernetes/pki/ 
[root@master1 ~] scp /etc/kubernetes/pki/sa.pub master2:/etc/kubernetes/pki/ 
[root@master1 ~] scp /etc/kubernetes/pki/front-proxy-ca.crt master2:/etc/kubernetes/pki/ 
[root@master1 ~] scp /etc/kubernetes/pki/front-proxy-ca.key master2:/etc/kubernetes/pki/ 
[root@master1 ~] scp /etc/kubernetes/pki/etcd/ca.crt master2:/etc/kubernetes/pki/etcd/ 
[root@master1 ~] scp /etc/kubernetes/pki/etcd/ca.key master2:/etc/kubernetes/pki/etcd/
```

**证书拷贝之后在 master2 上执行如下命令，这样就可以把master2 和加入到集群，成为控制节点：**

````bash
# 在master1 上查看加入节点的命令： 

[root@master1 ~] kubeadm token create --print-join-command

# 在生成token后面添加两个条件，然后在master2上执行：
```bash
kubeadm join 192.168.179.199:16443 --token mak56b.och6ggjdkih3y0oj     --discovery-token-ca-cert-hash sha256:6162c7fbbca43378af14677c88083d93999f7543555ba60e9587b915a06859f8 --control-plane --ignore-preflight-errors=SystemVerification
```
# 再执行
[root@master2 ~] mkdir -p $HOME/.kube 
[root@master2 ~] sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config 
[root@master2 ~] sudo chown $(id -u):$(id -g) $HOME/.kube/config 
[root@master2 ~] kubectl get nodes
````

![image-20250329160601622](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250329160601622.png)

![image-20250329160823093](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250329160823093.png)

**看到上面说明 master2 节点已经加入到集群了**

#### （7）**扩容 k8s 集群-添加 node 节点**

```bash
在 master1 上查看加入节点的命令： 
[root@master1 ~] kubeadm token create --print-join-command 
#显示如下： 
kubeadm join 192.168.179.199:16443 --token cc8i0a.xvejrdyo60s1c95o     --discovery-token-ca-cert-hash sha256:6162c7fbbca43378af14677c88083d93999f7543555ba60e9587b915a06859f8
把 node1 加入 k8s 集群： 
[root@node1~] kubeadm join 192.168.179.199:16443 --token cc8i0a.xvejrdyo60s1c95o     --discovery-token-ca-cert-hash sha256:6162c7fbbca43378af14677c88083d93999f7543555ba60e9587b915a06859f8 --ignore-preflight-errors=SystemVerification
```

![image-20250329161508359](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250329161508359.png)

```bash
#可以看到 node1 的 ROLES 角色为空，<none>就表示这个节点是工作节点。 
#可以把 node1 的 ROLES 变成 work，按照如下方法： 
[root@master1 ~] kubectl label node node1 node-role.kubernetes.io/worker=worker
```

**注意：上面状态都是 NotReady 状态，说明没有安装网络插件**

```bash
[root@master1 ~] kubectl get pods -n kube-system

[root@master1 ~] kubectl get pods -n kube-system -o wide
```

![image-20250329161912223](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250329161912223.png)

coredns-7f89b7bc75-dq7km是 pending 状态，这是因为还没有安装网络插件，等到下面安装好网络插件之后这个 cordns 就会变成 running 了

#### （8）**安装 kubernetes 网络组件-Calico**

**上传 calico.yaml 到 master1 上，使用 yaml 文件安装 calico 网络插件 。**

```bash
[root@master1 ~] kubectl apply -f calico.yaml

[root@master1 ~] kubectl get pod -n kube-system
```

![image-20250329162609234](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250329162609234.png)

**coredns-这个 pod 现在是 running 状态，运行正常**

![image-20250329162702412](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250329162702412.png)

**STATUS 状态是 Ready，说明 k8s 集群正常运行了**

#### （9）测试在 k8s 创建 pod 是否可以正常访问网络

```bash
#把 busybox-1-28.tar.gz 上传到 node1 节点，手动解压 
[root@node1 ~] docker load -i busybox-1-28.tar.gz 
[root@master1 ~] kubectl run busybox --image busybox:1.28 --restart=Never --rm -it busybox -- sh
/ # ping www.baidu.com
PING www.baidu.com (39.156.66.18): 56 data bytes
64 bytes from 39.156.66.18: seq=0 ttl=127 time=39.3 ms
# 通过上面可以看到能访问网络，说明 calico 网络插件已经被正常安装了
```

#### （10）**测试 k8s 集群中部署 tomcat 服务**

````bash
# 把 tomcat.tar.gz 上传到 node1，手动解压
[root@node1 ~] docker load -i tomcat.tar.gz
[root@master1 ~] kubectl apply -f tomcat.yaml
[root@master1 ~] kubectl get pods 
```bash
NAME READY STATUS RESTARTS AGE 
demo-pod 1/1 Running 0 10s 
```
[root@master1 ~] kubectl apply -f tomcat-service.yaml
[root@master1 ~] kubectl get svc
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
kubernetes ClusterIP 10.255.0.1 <none> 443/TCP 158m
tomcat NodePort 10.255.227.179 <none> 8080:30080/TCP 19m
在浏览器访问 xianchaonode1 节点的 ip:30080 即可请求到浏览器
````

#### (11)**测试 coredns 是否正常**

````bash
[root@master1 ~] kubectl run busybox --image busybox:1.28 --restart=Never --rm -it busybox -- sh 

If you don't see a command prompt, try pressing enter. 

/ # nslookup kubernetes.default.svc.cluster.local 
```bash
Server:    10.10.0.10
Address 1: 10.10.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes.default.svc.cluster.local
Address 1: 10.10.0.1 kubernetes.default.svc.cluster.local 
```
/ # nslookup tomcat.default.svc.cluster.local 
```bash
Server:    10.10.0.10
Address 1: 10.10.0.10 kube-dns.kube-system.svc.cluster.local

Name:      tomcat.default.svc.cluster.local
Address 1: 10.10.0.80 tomcat.default.svc.cluster.local
```
10.10.13.88 就是我们 coreDNS 的 clusterIP，说明 coreDNS 配置好了。
解析内部 Service 的名称，是通过 coreDNS 去解析的。 
 
10.10.0.10 是创建的 tomcat 的 service ip
````

至此，k8s集群搭建完成。
