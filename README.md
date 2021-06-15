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
