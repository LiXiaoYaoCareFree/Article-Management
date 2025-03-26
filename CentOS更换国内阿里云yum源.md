## CentOS更换国内阿里云yum源

### 1.确保虚拟机已经联网

```bash
ping www.baidu.com
```

![image-20250325144107367](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250325144107367.png)

出现以上输出代表已经联网。

### 2.备份现有yum配置文件

```bash
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
```

- /etc/yum.repos.d/CentOS-Base.repo 是yum的配置文件
- /etc/yum.repos.d/CentOS-Base.repo.bak 是我们备份的配置文件

### 3.下载阿里云yum源

我这里的系统是 `CentOS7`，所以下的是 `Centos-7.repo`,可以根据自己的系统版本选择。

```bash
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
```

### 4.清理缓存

```bash
yum clean all
```

`yum clean all` 命令用于清理 `YUM (Yellowdog Updater Modified)` 包管理器的所有缓存。当你执行这个命令时，YUM 会清除所有已下载的元数据和软件包缓存。

### 5.重新生成缓存

```bash
yum makecache
```

当你运行 yum makecache 命令时，YUM 会执行以下操作：

检查配置文件：YUM 会读取配置文件，通常位于 /etc/yum.repos.d/ 目录下，这些文件定义了可用的软件仓库。
生成缓存：YUM 会为每个配置好的仓库生成一个缓存。这涉及到从每个仓库的元数据服务器下载必要的信息，例如软件包列表、版本等，并将其存储在本地文件系统上（默认位置通常是 /var/cache/yum/）。

### 6.测试安装

```bash
yum install gcc
```

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/f36900fafac44e12b5fe468616865c9c.jpeg#pic_center)

如图显示则安装完毕。