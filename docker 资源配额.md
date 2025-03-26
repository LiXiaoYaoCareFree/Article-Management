## docker 资源配额

### docker容器控制cpu

Docker 通过 cgroup 来控制容器使用的资源限制，可以对 docker 限制的资源包括 CPU、内存、磁盘。

`docker run --help | grep cpu-shares`

cpu 配额参数：`-c, --cpu-shares int`

CPU shares (relative weight) 在创建容器时指定容器所使用的 CPU 份额值。cpu-shares 的值不能

保证可以获得 1 个 vcpu 或者多少 GHz 的 CPU 资源，仅仅只是一个弹性的加权值。

默认每个 docker 容器的 cpu 份额值都是 1024。在同一个 CPU 核心上，同时运行多个容器时，容器的

cpu 加权的效果才能体现出来。



例： 两个容器 A、B 的 cpu 份额分别为 1000 和 500，结果会怎么样？

 

情况 1：A 和 B 正常运行，占用同一个 CPU，在 cpu 进行时间片分配的时候，容器 A 比容器 B 多一倍

的机会获得 CPU 的时间片。 

 

情况 2：分配的结果取决于当时其他容器的运行状态。比如容器 A 的进程一直是空闲的，那么容器 B

是可以获取比容器 A 更多的 CPU 时间片的； 比如主机上只运行了一个容器，即使它的 cpu 份额只有

50，它也可以独占整个主机的 cpu 资源。

 

cgroups 只在多个容器同时争抢同一个 cpu 资源时，cpu 配额才会生效。因此，无法单纯根据某个容

器的 cpu 份额来确定有多少 cpu 资源分配给它，资源分配结果取决于同时运行的其他容器的 cpu 分

配和容器中进程运行情况。

例 1：给容器实例分配 512 权重的 cpu 使用份额 

参数： --cpu-shares 512 

`docker run -it --cpu-shares 512 centos /bin/bash `

` cat /sys/fs/cgroup/cpu/cpu.shares`

 查看结果： 

512

 

注：稍后，我们启动多个容器，测试一下是不是只能使用 512 份额的 cpu 资源。单独一个容器，看

不出来使用的 cpu 的比例。 因没有 docker 实例同此 docker 实例竞争。

**总结：**

**通过-c 设置的 cpu share 并不是 CPU 资源的绝对数量，而是一个相对的权重值。某个容器最终能**

**分配到的 CPU 资源取决于它的 cpu share 占所有容器 cpu share 总和的比例。通过 cpu share**

**可以设置容器使用 CPU 的优先级。**



比如在 host 中启动了两个容器：

docker run --name "container_A" -c 1024 ubuntu

docker run --name "container_B" -c 512 ubuntu



container_A 的 cpu share 1024，是 container_B 的两倍。当两个容器都需要 CPU 资源时，

container_A 可以得到的 CPU 是 container_B 的两倍。

需要注意的是，这种按权重分配 CPU 只会发生在 CPU 资源紧张的情况下。如果 container_A 处于空

闲状态，为了充分利用 CPU 资源，container_B 也可以分配到全部可用的 CPU。

### CPU core 核心控制

参数：--cpuset 可以绑定 CPU 

对多核 CPU 的服务器，docker 还可以控制容器运行限定使用哪些 cpu 内核和内存节点，即使用--

cpuset-cpus 和--cpuset-mems 参数。对具有 NUMA 拓扑（具有多 CPU、多内存节点）的服务器尤其有

用，可以对需要高性能计算的容器进行性能最优的配置。如果服务器只有一个内存节点，则--

cpuset-mems 的配置基本上不会有明显效果。

 

扩展： 

服务器架构一般分： SMP、NUMA、MPP 体系结构介绍 

从系统架构来看，目前的商用服务器大体可以分为三类： 

1. 即对称多处理器结构(SMP ： Symmetric Multi-Processor) 例： x86 服务器，双路服务

器。主板上有两个物理 cpu 

2. 非一致存储访问结构 (NUMA ： Non-Uniform Memory Access) 例： IBM 小型机

pSeries 690 

3. 海量并行处理结构 (MPP ： Massive ParallelProcessing) 。 例： 大型机 Z14 

 

### CPU 配额控制参数的混合使用

在上面这些参数中，cpu-shares 控制只发生在容器竞争同一个 cpu 的时间片时有效。 

如果通过 `cpuset-cpus` 指定容器 A 使用 cpu 0，容器 B 只是用 cpu1，在主机上只有这两个容器使用

对应内核的情况，它们各自占用全部的内核资源，cpu-shares 没有明显效果。 

如何才能有效果？

**容器 A 和容器 B 配置上 `cpuset-cpus` 值并都绑定到同一个 cpu 上，然后同时抢占 cpu 资源，**

**就可以看出效果了。** 

例 1：测试 cpu-shares 和 cpuset-cpus 混合使用运行效果，就需要一个压缩力测试工具 stress 来让

容器实例把 cpu 跑满。 

如何把 cpu 跑满？

如何把 4 核心的 cpu 中第一和第三核心跑满？可以运行 stress，然后使用 taskset 绑定一下 cpu。 

 

先扩展：stress 命令 

概述：linux 系统压力测试软件 Stress 。

 **`yum install -y epel-release `**

**`yum install stress -y `**

例 1：产生 2 个 cpu 进程，2 个 io 进程，20 秒后停止运行 

**`stress -c 2 -i 2 --verbose --timeout 20s `**

如果执行时间为分钟，改 20s 为 1m 

查看：

`top`

例 1：测试 cpuset-cpus 和 cpu-shares 混合使用运行效果，就需要一个压缩力测试工具 stress 来让

容器实例把 cpu 跑满。 当跑满后，会不会去其他 cpu 上运行。 如果没有在其他 cpu 上运行，说明

cgroup 资源限制成功。 



实例 3：创建两个容器实例:docker10 和 docker20。 让 docker10 和 docker20 只运行在 cpu0 和

cpu1 上，最终测试一下 docker10 和 docker20 使用 cpu 的百分比。实验拓扑图如下：

运行两个容器实例 

`docker run -itd --name docker10 --cpuset-cpus 0,1 --cpu-shares 512 centos /bin/bash `

- 指定 docker10 只能在 cpu0 和 cpu1 上运行，而且 docker10 的使用 cpu 的份额 512 

- 参数-itd 就是又能打开一个伪终端，又可以在后台运行着 docker 实例 

`docker run -itd --name docker20 --cpuset-cpus 0,1 --cpu-shares 1024 centos /bin/bash `

**指定 docker20 只能在 cpu0 和 cpu1 上运行，而且 docker20 的使用 cpu 的份额 1024，**

**比 dcker10 多一倍 **

 

测试 1： 进入 docker10，使用 stress 测试进程是不是只在 cpu0,1 上运行： 

` docker exec -it docker10 /bin/bash `

`yum install -y epel-release ` #安装 epel 扩展源 

` yum install stress -y` #安装 stress 命令 

`stress -c 2 -v -t 10m` #运行 2 个进程，把两个 cpu 占满 

在物理机另外一个虚拟终端上运行 top 命令，按 1 快捷键，查看每个 cpu 使用情况： 

 

可看到正常。只在 cpu0,1 上运行 

 

测试 2： 然后进入 docker20，使用 stress 测试进程是不是只在 cpu0,1 上运行，且 docker20 上运行

的 stress 使用 cpu 百分比是 docker10 的 2 倍 

`docker exec -it docker20 /bin/bash `

`yum install -y epel-release` #安装 epel 扩展源 

` yum install stress -y`

` stress -c 2 -v -t 10m `

在另外一个虚拟终端上运行 top 命令，按 1 快捷键，查看每个 cpu 使用情况： 

 

注：两个容器只在 cpu0,1 上运行，说明 cpu 绑定限制成功。而 docker20 是 docker10 使用 cpu 的 2

倍。说明--cpu-shares 限制资源成功。 

 

### docker 容器控制内存

Docker 提供参数-m, --memory=""限制容器的内存使用量。 

例 1：允许容器使用的内存上限为 128M： 

`docker run -it -m 128m centos `

查看： 

` cat /sys/fs/cgroup/memory/memory.limit_in_bytes `

134217728 

注：也可以使用 tress 进行测试，到现在，我可以限制 docker 实例使用 cpu 的核心数和权重，可以

限制内存大小。 

例 2：创建一个 docker，只使用 2 个 cpu 核心，只能使用 128M 内存 

[root@xianchaomaster1 ~]# docker run -it --cpuset-cpus 0,1 -m 128m centos 

 

### docker 容器控制 IO

`docker run --help | grep write-b` 

限制此设备上的写速度（bytes per second），单位可以是 kb、mb 或者 gb。 

`--device-read-bps value` #限制此设备上的读速度（bytes per second），单位可以是 kb、mb 或

者 gb。

情景：防止某个 Docker 容器吃光你的磁盘 I / O 资源

例 1：限制容器实例对硬盘的最高写入速度设定为 2MB/s。 

--device 参数：将主机设备添加到容器 

`mkdir -p /var/www/html/`

` docker run -it -v /var/www/html/:/var/www/html --device  /dev/sda:/dev/sda --device-write-bps /dev/sda:2mb centos /bin/bash`

` time dd if=/dev/sda of=/var/www/html/test.out bs=2M count=50  oflag=direct,nonblock`

注：dd 参数： 

direct：读写数据采用直接 IO 方式，不走缓存。直接从内存写硬盘上。 

nonblock：读写数据采用非阻塞 IO 方式，优先写 dd 命令的数据 

50+0 records in 

50+0 records out 

52428800 bytes (52 MB) copied, 50.1831 s, 2.0 MB/s 

 

real 0m50.201s 

user 0m0.001s 

sys 0m0.303s 

注： 发现 1 秒写 2M。 限制成功。

 

### docker 容器运行结束自动释放资源

**`docker run --help | grep rm `**

 `--rm` 参数： Automatically remove the container when it exits 

作用：当容器命令运行结束后，自动删除容器，自动释放资源 

**例：** 

`docker run -it --rm --name lly centos sleep 6 `

物理上查看： 

`docker ps -a | grep lly` 

`6c75a9317a6b 	centos	 "sleep 6"	 6 seconds ago 	Up 4  seconds 	mk `

等 5s 后，再查看： 

**`docker ps | grep lly` **

自动删除了。