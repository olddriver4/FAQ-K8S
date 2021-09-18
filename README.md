### K8S
- 压测遇到cpu throttling问题 resources - requests 和 limits
> 这是一个 Linux 内核的 Bug，他会对设置了 CPU 限制的容器进行不必要的流控。 升级4.19 或更高版本的 Linux 发行版已经纠正了该问题

- 处理应用假死(端口正常，api无法访问)
>建议：readinessprobe七层api接口探测，livenessprobe四层tcp端口探测；假死情况服务存在个别bug或者特殊服务，需要和研发确定，livenessprobe建议启动七层探测

- 启动特别慢(需要3m)。是要深入了解pod的生命周期和容器探针
> 启动很慢的服务需要消耗很长的时间，否则pod会抛出CrashLoopBackOff错误，需要配合研发深入了解服务运行时效，结合探针监控存货

- 发现某个node节点，上面内存使用了50%，另外的使用了80%，极度不均衡
> 建议使用Descheduler实现二次调度，主要针对 增减node时，分配不均匀的情况；内部流程：变更 - 打分 - 过滤 - 调度

- azure k8s  弹性扩容的node，网络无法拉取外网镜像
> 使用azure弹性扩容的node外网不通，做了抓包和ping测试，需要重启node服务器解决， 已和工单反馈确认问题。

- nodeAffinity podAffinity ，node、pod亲和性；podAnitAffinity反亲和性。
> 在schedule调度过程中，如果目标使用亲和性策略 但是目标亲和性服务器资源不足当前分配的内存和cpu数，则pod生命周期就会报CrashLoopBackOff 无法成功添加工作节点的问题。 最好配合PodDisruptionBudget（PDB）开启自我保护措施  

- k8s 中出现了大量的 Evction  
> 由于node节点资源不足，造成资源的争抢，并出现大量的驱逐的pod，找出Evicted pod 节点并删除pod：cat /var/paas/sys/log/kubernetes/kubelet.log | grep -i Evicted -C3


### Docker
-  docker-compose  1.17以下版本无法正确的限制docker内存和cpu使用，docker stats还是无限制状态
> 这是一个Docker Bug，已提交issue，之后版本无此类问题

- docker stop 无法优雅停止，指定-t也超时
>首先Linux版本bash方式 ubuntu下：/bin/sh -> dash，在centos下：/bin/sh -> bash。CMD或者ENTRYPOINT （command param1 param2） 使用shell 格式，默认/bin/sh -c ，要是用["executable", "param1", "param2"]   exec 使用 就没问题，启动的时候如果 CMD带了变量 ，需要指定SHELL ["/bin/bash", "-c"]。 还需程序做好信号处理。

- 网络问题，无法pull和push
> 使用阿里云或者其他第三方代理等

- 容器运行多个程序需求
> 最好使用supervisord ，但不建议一个容器运行多个进程

- 使用内存和 swap 限制启动容器时候报警告："WARNING: Your kernel does not support cgroup swap limit. WARNING: Your kernel does not support swap limit capabilities. Limitation discarded."？
> 这是因为系统默认没有开启对内存和 swap 使用的统计功能，引入该功能会带来性能的下降。要开启该功能，可以采取如下操作：
		编辑 /etc/default/grub 文件（Ubuntu 系统为例），配置 GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1"
		更新 grub：$ sudo update-grub
		重启系统，即可。

-  docker 缓存问题
> 先docker build --no-cache
   在执行 docker stop 在执行docker-compose up

- docker-compose healthcheck和depends_on 检查顺利启动不依赖问题
>需要在dockefile添加 ENTRYPOINT /go/wait-for.sh 127.0.0.1:22530 -- run 
 检查需要依赖的主程序22530是否启动成功，然后在启动本身就不会报错了。
 wai-for.sh URL：https://github.com/vishnubob/wait-for-it
 
### Redis
- Redis内存溢出问题
> 因为开发使用的Redis 没有设置ttl过期，而这块设置的maxmemory内存限制触发的定期删除策略无用，主动删除策略配置的是存在ttl过期缓存使用少的删除， 因此 联系开发让其配置ttl过期策略，主动策略配置为allkeys-lfu
- Redis CPU百分百问题  
> redis 开启aof 每秒持久化数据落盘本地 并开启了重写功能，在高并发的写入数据时，由于redis默认CPU但进程，导致cpu百分百，aof从内存落盘到本地跟不上实际 写入时间，redis无法访问。
> 优化：redis 6.0后支持开启多线程， infura数据可以容忍数据短暂丢失，关闭aof持久化，后续继续跟进
> “实际集群环境中，redis备份rdb aof策略需要单独一台与业务无关的从服务器进行备份操作”

### PgPool
- pgpool客户端阻塞  

>   最近遇到一个PgPool连接阻塞问题，PgPool刚开启是能成功连接的，过段时间就连接不上了。查看PgPool日志，启动成功，连接数据库节点成功，健康检查成功。然后怀疑是并发数过多导致阻塞。
>一开始，更改了pgpool.conf的max_pool,num_init_children参数然后重启，结果仍然阻塞。查资料可知：
>num_init_children：pgPool允许的最大并发数，默认32。
>max_pool：连接池的数量，默认4。
>pgpool需要的数据库连接数=num_init_children*max_pool；
>后检查Postgresql数据库的postgresql.conf文件的max_connections=100
>superuser_reserved_connections=3。
>pgpool的连接参数应当满足如下公式：
>num_init_children*max_pool<max_connections-superuser_reserved_connections

当需要pgpool支持更多的并发时，需要更改num_init_children参数，同时要检查下num_init_children*max_pool是否超过了max_connections-superuser_reserved_connections，如果超过了，可将max_connections改的更大。
