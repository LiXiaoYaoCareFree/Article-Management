## docker私有仓库harbor

Harbor 介绍

Docker 容器应用的开发和运行离不开可靠的镜像管理，虽然 Docker 官方也提供了公共的镜像仓库，

但是从安全和效率等方面考虑，部署我们私有环境内的 Registry 也是非常必要的。Harbor 是由 VMware

公司开源的企业级的 Docker Registry 管理项目，它包括权限管理(RBAC)、LDAP、日志审核、管理界面、

自我注册、镜像复制和中文支持等功能。

### 为 Harbor 自签发证书

`hostnamectl set-hostname harbor && bash`

`mkdir /data/ssl -p`

`cd /data/ssl/`

#### **生成 ca 证书**

**生成一个 3072 位的 key，也就是私钥**

`openssl genrsa -out ca.key 3072`

**生成一个数字证书 ca.pem，3650 表示证书的有效时间是 3 年，按箭头提示填写即可，没有箭头标注的为空**

`openssl req -new -x509 -days 3650 -key ca.key -out ca.pem`

![image-20250326222743062](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250326222743062.png)

#### **生成域名的证书**

**生成一个 3072 位的 key，也就是私钥**

`openssl genrsa -out harbor.key 3072`

**生成一个证书请求，一会签发证书时需要的，标箭头的按提示填写，没有箭头标注的为空**

`openssl req -new -key harbor.key -out harbor.csr`

![image-20250326222758015](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250326222758015.png)

#### **签发证书**

`openssl x509 -req -in harbor.csr -CA ca.pem -CAkey ca.key -CAcreateserial -out harbor.pem -days 3650`

显示如下，说明证书签发好了：

![image-20250326222937487](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250326222937487.png)

## **安装 Harbor**

**创建安装目录**

`mkdir /data/install -p`

`cd /data/install/`

**安装 harbor**

/data/ssl 目录下有如下文件：

`ca.key ca.pem ca.srl harbor.csr harbor.key harbor.pem`

把 harbor 的离线包 harbor-offline-installer-v2.3.0-rc3.tgz 上传到这个目录

解压:

```bash
tar zxvf harbor-offline-installer-v2.3.0-rc3.tgz

cd harbor

cp harbor.yml.tmpl harbor.yml

vim harbor.yml
```

修改配置文件：

`hostname: harbor `

\#修改 hostname，跟上面签发的证书域名保持一致

\#协议用 https

`certificate: /data/ssl/harbor.pem`

`private_key: /data/ssl/harbor.key`

邮件和 ldap 不需要配置，在 harbor 的 web 界面可以配置

其他配置采用默认即可

修改之后保存退出

注：harbor 默认的账号密码：`admin/Harbor12345`

## 安装 docker-compose

上传 docker-compose-Linux-x86_64 文件到 harbor 机器

```bash
mv docker-compose-Linux-x86_64.64 /usr/bin/docker-compose

chmod +x /usr/bin/docker-compose
```

**注： docker-compose 项目是 Docker 官方的开源项目，负责实现对 Docker 容器集群的快速编排。**

**Docker-Compose 的工程配置文件默认为 docker-compose.yml，Docker-Compose 运行目录下的必要有一个docker-compose.yml。docker-compose 可以管理多个 docker 实例。**

安装 harbor 需要的离线镜像包 docker-harbor-2-3-0.tar.gz ，可上传到 harbor 机器，通过docker load -i 解压:

`docker load -i docker-harbor-2-3-0.tar.gz`

`cd /data/install/harbor`

`./install.sh`

![image-20250326231833781](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250326231833781.png)

在 hosts 文件添加如下一行，然后保存即可

`192.168.40.181 harbor`



**扩展：**

如何停掉 harbor：

```bash
cd /data/install/harbor

docker-compose stop 
```

如何启动 harbor：

```bash
cd /data/install/harbor

docker-compose start
```

如果 docker-compose start 启动 harbor 之后，还是访问不了，那就需要重启虚拟机

**Harbor 图像化界面使用说明** 

在浏览器输入：

https://harbor

## **测试使用 harbor 私有镜像仓库** 

**修改 docker 配置**

`vim /etc/docker/daemon.json`

```json
{ "registry-mirrors": ["https://rsbud4vc.mirror.aliyuncs.com","https://registry.docker-cn.com","https://docker.mirrors.ustc.edu.cn","https://dockerhub.azk8s.cn","http://hub-mirror.c.163.com"],"insecure-registries":  ["192.168.179.140","harbor"]
}
```

```bash
# 修改配置之后使配置生效：

systemctl daemon-reload && systemctl restart docker

#查看 docker 是否启动成功

systemctl status docker
```

显示如下，说明启动成功：

`Active: active (running) since Fri … ago`

注意：

配置新增加了一行内容如下：

"insecure-registries":["192.168.179.140"], 

上面增加的内容表示我们内网访问 harbor 的时候走的是 http，192.168.179.140 是安装 harbor 机器的 ip。

**登录 harbor：**

`docker login 192.168.179.140`

`Username：admin`

`Password: Harbor12345`

输入账号密码之后看到如下，说明登录成功了：

Login Succeeded

导入 tomcat 镜像，tomcat.tar.gz

**`docker load -i tomcat.tar.gz`**

\#把 tomcat 镜像打标签

**`docker tag tomcat:latest 192.168.179.140/test/tomcat:v1`**

执行上面命令就会把 192.168.40.181/test/tomcat:v1 上传到 harbor 里的 test 项目下

**`docker push 192.168.179.140/test/tomcat:v1`**

执行上面命令就会把 192.168.179.140/test/tomcat:v1 上传到 harbor 里的 test 项目下

**从 harbor 仓库下载镜像** 

在 CentOS 7 机器上删除镜像

`docker rmi -f 1192.168.179.140/test/tomcat:v1`

拉取镜像

`docker pull 192.168.179.140/test/tomcat:v1`