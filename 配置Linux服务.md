## 配置Linux服务

### 1.网络配置

```bash
vi /etc/sysconfig/network-scripts/ifcfg-ens33
```

配置静态IP：

```bash
# 修改BOOTPROTO
BOOTPROTO="static"

# 添加静态IP
IPADDR=192.168.179.139
GATEWAY=192.168.179.2
DNS1=192.168.179.2
```

重启服务：

```bash
service network restart
```

### 2.CentOS换源

```bash
ping www.baidu.com

mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak

wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

yum clean all

yum makecache

yum install gcc
```



### 3.配置主机名

```bash
hostnamectl set-hostname master && bash
```

### 4.关闭防火墙

```bash
systemctl stop firewalld && systemctl disable firewalld
```

### 5.关闭iptables防火墙

```bash
# 安装iptables
yum install iptables-services -y

# 禁用iptables
service iptables stop   && systemctl disable iptables

# 清空防火墙规则
iptables -F
```

### 6.关闭selinux

```bash
setenforce 0

# 注意：修改selinux配置文件之后，重启机器，selinux才能永久生效
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

# 重启服务
reboot

# 显示Disabled表示selinux关闭成功
getenforce 
```

### 7.配置时间同步

```bash
yum install -y ntp ntpdate

ntpdate cn.pool.ntp.org

# 编写计划任务保证时间同步
crontab -e
```

**添加配置**

```bash
* */1 * * * /usr/sbin/ntpdate cn.pool.ntp.org
```
**重启crond服务使配置生效**

`systemctl restart crond`

### 8.安装基础软件包

```bash
yum install -y  wget net-tools nfs-utils lrzsz gcc gcc-c++ make cmake libxml2-devel openssl-devel curl curl-devel unzip sudo ntp libaio-devel wget vim ncurses-devel autoconf automake zlib-devel python-devel epel-release openssh-server socat ipvsadm conntrack 
```

### 9.安装docker-ce

```bash
# 配置docker-ce国内yum源（阿里云）
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 安装docker依赖包
yum install -y yum-utils device-mapper-persistent-data lvm2

# 安装docker-ce
yum install docker-ce -y

# 启动docker服务
systemctl start docker && systemctl enable docker 
systemctl status docker
```

### 10.开启包转发功能和修改内核参数

内核参数修改：br_netfilter模块用于将桥接流量转发至iptables链，br_netfilter内核参数需要开启转发。

````bash
modprobe br_netfilter

# 配置docker.conf
# cat > /etc/sysctl.d/docker.conf << EOF 
# net.bridge.bridge-nf-call-ip6tables = 1
# net.bridge.bridge-nf-call-iptables = 1
# net.ipv4.ip_forward = 1
# EOF
vim /etc/sysctl.d/docker.conf
```bash
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
```

# 使参数生效
sysctl -p /etc/sysctl.d/docker.conf

# 重启后模块失效，下面是开机自动加载模块的脚本
# 在/etc/新建rc.sysinit 文件
vim /etc/rc.sysinit

```bash
#!/bin/bash
for file in /etc/sysconfig/modules/*.modules ; do [ -x $file ] && $file
done
```

# 在/etc/sysconfig/modules/目录下新建文件如下
vim /etc/sysconfig/modules/br_netfilter.modules

```bash
modprobe br_netfilter
```

# 增加权限
chmod 755 /etc/sysconfig/modules/br_netfilter.modules

# 重启机器模块也会自动加载
lsmod |grep br_netfilter

# 重启docker
systemctl restart docker
````

**注：**

Docker安装后出现 WARNING: bridge-nf-call-iptables is disabled 的解决办法：

`net.bridge.bridge-nf-call-ip6tables = 1`

`net.bridge.bridge-nf-call-iptables = 1`

`net.ipv4.ip_forward = 1`

**将Linux系统作为路由或者VPN服务就必须要开启IP转发功能。当linux主机有多个网卡时一个网卡收到的信息是否能够传递给其他的网卡，如果设置成1 的话可以进行数据包转发，可以实现VxLAN 等功能。不开启会导致docker部署应用无法访问。**



### 11.docker配置代理

````bash
# 配置代理网址
https://tagxx.vip/

# 创建文件夹docker.service.d
mkdir /etc/systemd/system/docker.service.d

# 创建配置文件
vim /etc/systemd/system/docker.service.d/http-proxy.conf

```bash
[Service]
Environment="HTTP_PROXY=http://本机ip地址:代理端口号"
Environment="HTTPS_PROXY=http://本机ip地址:代理端口号"
```

# 然后重新加载配置并重启服务
systemctl daemon-reload
systemctl restart docker

# 然后检查加载的配置
systemctl show docker --property Environment
````

### 12.VMware虚拟机共享宿主机网络资源

由于需要在 Linux 环境下进行一些测试工作，于是决定使用 VMware 虚拟化软件来安装 Ubuntu 24.04 .1操作系统。考虑到测试过程中需要访问 Github ，要使用Docker拉去镜像等外部网络资源，因此产生了让 Ubuntu 虚拟机共享宿主机（即运行 VMware 的物理机器）DL设置的需求。

虚拟机网络模式：NAT模式

请先打开Clash中Tun(虚拟网卡)模式

````bash
# 打开 ~/.bashrc 文件
vim ~/.bashrc

# 将宿主机IP地址和端口号替换为实际的值。
# 在文件的末尾添加以下两行内容，分别设置 HTTP 和 HTTPS DL：
```bash
export http_proxy=http://宿主机IP地址:端口号
export https_proxy=https://宿主机IP地址:端口号
```

# 是配置生效
source ~/.bashrc

# 测试 
curl -I google.com
````