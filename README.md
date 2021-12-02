
K8s node image过多问题（使用Kubelet垃圾回收）
kubelet每隔1分钟清理一次容器，5分钟清理一次镜像，代码中固化，不可自定义配置（v1.18+）
问题：node节点硬盘不足
参考：https://v1-17.docs.kubernetes.io/zh/docs/concepts/cluster-administration/kubelet-garbage-collection/

K8s滚动更新出现中断
参考：https://www.kubernetes.org.cn/7714.html
Deployment + LoadBalancer + Service引发中断问题,Deployment 滚动更新时会先创建新 pod，等待新 pod running 后再删除旧 pod。
问题_1：存在 Pod 收到 SIGTERM 信号并且停止工作后，还未从 Endpoints 中移除的情况，此时，请求从 slb 转发到 pod 中，而 Pod 已经停止工作，因此会出现服务中断
解决_1：为 pod 配置 preStop Hook，使 Pod 收到 SIGTERM 时 sleep 一段时间而不是立刻停止工作，从而确保从 SLB 转发的流量还可以继续被 Pod 处理

问题_2：当 pod 变为 termintaing 状态时，会从所有 service 的 endpoint 中移除该 pod。kube-proxy 会清理对应的 iptables/ipvs 条目。而容器服务 watch 到 endpoint 变化后，会调用 alb openapi 移除后端，此操作会耗费几秒。由于这两个操作是同时进行，因此有可能存在节点上的 iptables/ipvs 条目已经被清理，但是节点还未从 alb 移除的情况。此时，流量从 alb 流入，而节点上已经没有对应的 iptables/ipvs 规则导致服务中断
解决_2：local模式 绕过svc 直接代理到alb，cluster模式多个pod 分布在不同的node（建议）

问题_3：长连接中断，容器服务监控到 Endpoints 变化后，会将 Node 从 alb 后端移除。当节点从 alb 后端移除后，ALB 对于继续发往该节点的长连接会直接断开，导致服务中断
解决_3：为 ALB 设置长链接优雅中断（依赖具体云厂商）

压测遇到cpu throttling问题 resources - requests 和 limits
这是一个 Linux 内核的 Bug，他会对设置了 CPU 限制的容器进行不必要的流控。 升级4.19 或更高版本的 Linux 发行版已经纠正了该问题

处理应用假死(端口正常，api无法访问)
建议：readinessprobe七层api接口探测，livenessprobe四层tcp端口探测；假死情况服务存在个别bug或者特殊服务，需要和研发确定，livenessprobe建议启动七层探测

启动特别慢(需要3m)。是要深入了解pod的生命周期和容器探针
启动很慢的服务需要消耗很长的时间，否则pod会抛出CrashLoopBackOff错误，需要配合研发深入了解服务运行时效，结合探针监控存货

nodeAffinity podAffinity ，node、pod亲和性；podAnitAffinity反亲和性。
在schedule调度过程中，如果目标使用亲和性策略 但是目标亲和性服务器资源不足当前分配的内存和cpu数，则pod生命周期就会报CrashLoopBackOff 无法成功添加工作节点的问题。 最好配合PodDisruptionBudget（PDB）开启自我保护措施

发现某个node节点，上面内存使用了50%，另外的使用了80%，极度不均衡
建议使用Descheduler实现二次调度，主要针对 增减node时，分配不均匀的情况；内部流程：变更 - 打分 - 过滤 - 调度

k8s 中出现了大量的 Evction
cat /var/paas/sys/log/kubernetes/kubelet.log | grep -i Evicted -C3
由于node节点资源不足，造成资源的争抢，并出现大量的驱逐的pod，找出Evicted pod 节点并删除pod

K8s grpc负载不均衡问题
问题：gRPC 是基于 HTTP/2 的，多个请求在一个 TCP 连接上多路复用，一旦 ClusterIP 和某个 pod 建立了 gRPC 连接后，因为多路复用的缘故，所以后续其它请求也都会被转发给此 pod，结果其它 pod 则完全被忽略了。
解决：1.使用 istio：存在客户端 具有入侵性，但性能更好。
           2.使用headless service：无入侵性，但根据域名转发 性能打折扣，高并发有影响。

K8s “/var/lib/kubelet/pods”目录下的容器文件夹把空间占满了
参考：https://blog.espnlol.com/?p=708
问题：容器的log非常多，开发人员使用hostpath挂载卷，日志都写到了节点上，并且没有开启日志滚动和压缩，导致大量节点磁盘被占用
解决：hostpath挂载的卷在deployment和statefulsets删除后也依然存在，需要手动删除对应pod，迁移数据，修改pod yaml中hostpath配置变更为local pv
