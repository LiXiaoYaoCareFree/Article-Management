## Docker容器的数据管理

### 1.Docker 容器的数据卷

**什么是数据卷？** 

数据卷是经过特殊设计的目录，可以绕过联合文件系统（UFS），为一个或者多个容器提供访问，数据卷

设计的目的，在于数据的永久存储，它完全独立于容器的生存周期，因此，docker 不会在容器删除时删

除其挂载的数据卷，也不会存在类似的垃圾收集机制，对容器引用的数据卷进行处理，同一个数据卷可

以只支持多个容器的访问。

**数据卷的特点：** 

**1.数据卷在容器启动时初始化，如果容器使用的镜像在挂载点包含了数据，这些数据会被拷贝到新初始**

**化的数据卷中** 

**2.数据卷可以在容器之间共享和重用** 

**3.可以对数据卷里的内容直接进行修改** 

**4.数据卷的变化不会影像镜像的更新** 

**5.卷会一直存在，即使挂载数据卷的容器已经被删除**

### 2.数据卷的使用

1.为容器添加数据卷

**`docker run -v /datavolume:/data -it centos /bin/bash`**

如：

**`docker run --name volume -v ~/datavolume:/data -itd centos /bin/bash`** 

注：`~/datavolume `为宿主机目录，/data 为 docker 启动的 volume 容器的里的目录

这样在宿主机的`/datavolume` 目录下创建的数据就会同步到容器的/data 目录下

（1）为数据卷添加访问权限

**`docker run --name volume1 -v ~/datavolume1:/data:ro -itd centos /bin/bash`** 

添加只读权限之后在 docker 容器的/data 目录下就不能在创建文件了，为只读权限；在宿主机下的

`/datavolume1` 下可以创建东西。

2.使用 Dockerfile 构建包含数据卷的镜像

Dockerfile 指令：

**`volume[/data"]`**

创建dockerfile：

```dockerfile
FROM centos 
VOLUME ["/datavolume3","/datavolume6"] 
CMD /bin/bash
```

构建镜像：

**`docker build -t="volume" .`**

启动容器：

`docker run --name volume-doubble -it volume`

会看到这个容器下有两个目录，/datavolume3 和/datavolume6

## Docker的数据卷容器

什么是数据卷容器：

命名的容器挂载数据卷，其他容器通过挂载这个容器实现数据共享，挂载数据卷的容器，就叫做数据卷

容器。

**挂载数据卷容器的方法** 

**`docker run --volumes-from [container name]`**

**例：** 

**`docker run --name data-volume -itd volume`（volume 这个镜像是上面创建的带两个数据卷**

**/datavolume3 和/ddatavolume6 的镜像）**

**`docker exec -it data-volume /bin/bash`（进入到容器中）** 

**`touch /datavolume6/lucky.txt`** 

**退出容器 `exit`** 

**创建一个新容器挂载刚才 data-volume 这个容器创建的数据卷** 

**`docker run --name data-volume2 --volumes-from data-volume -itd centos /bin/bash`** 

**进入到新创建的容器** 

**`docker exec -it data-volume2 /bin/bash`** 

**查看容器的/datavolume6 目录下是否新创建了 lucky.txt 文件** 

**`cd /datavolume6`** 

**可以看见有刚才在上一个容器创建的文件 lucky.txt**

## Docker 数据卷的备份和还原

数据备份方法：

`docker run --volumes-from [container name] -v $(pwd):/backup centos tar czvf /backup/backup.tar [container data volume]`

**例子：** 

**`docker run --volumes-from data-volume2 -v /root/backup:/backup --name datavolume-copy centos tar zcvf /backup/data-volume2.tar.gz /datavolume6`**

 

**数据还原方法：** 

**`docker run --volumes-from [container name] -v $(pwd):/backup centos tar xzvf /backup/backup.tar.gz [container data volume]`** 

**例：** 

**`docker exec -it data-volume2 /bin/bash`** 

**`cd /datavolume6`** 

**`rm -rf lucky.txt`** 

**`docker run --volumes-from data-volume2 -v /root/backup/:/backup centos tar zxvf /backup/data-volume2.tar.gz -C /datavolume6`** 

**`docker exec -it data-volum2 /bin/bash`** 

**`cd /datavolum6`** 

可以看到还原后的数据。

