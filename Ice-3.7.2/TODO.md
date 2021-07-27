# 待分析的问题

1. Node 是如何判断服务进程是否存在，也就是如何检测进程异常退出
2. Registry 的 Master 和 slave 之间的关系
3. 服务的负载是如何统计的
4. Ice是如何交互数据的，RPC协议设计要点
5. Ice leader-follower 线程模式是如何实现的
6. 在 registry 处于 master-slave 模式下， client 是如何确定调用那个 locater 的。