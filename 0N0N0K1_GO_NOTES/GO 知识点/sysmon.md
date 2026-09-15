# 1. 介绍

在Golang中，**sysmon**是一个系统级的守护线程，它在程序启动时由Go Runtime创建。这个线程独立于GPM（Goroutine、Processor、Machine）模型之外，不需要P（Processor）就可以运行。

# 2. 功能

| 职责               | 说明                             | 涉及函数                  |
| ---------------- | ------------------------------ | --------------------- |
| 抢占长时间运行的 G       | 运行超 10ms 触发抢占（[[GMP]]）         | retake / preemptone   |
| 收回 syscall 阻塞的 P | 系统调用耗时过长则解绑 P                  | retake / handoffp     |
| 轮询网络事件           | 把就绪的 I/O 协程注入运行队列              | netpoll / injectglist |
| 强制执行 [[GC]]      | 超过 2 分钟没 GC 则触发                | forcegc               |
| 归还闲置内存           | GC 后闲置超 5 分钟的 span 还给 OS       | scavenger             |
| 检查死锁             | 程序无活动 goroutine 时判定 deadlocked | checkdead             |
| 处理到期 timer       | P 都在忙时兜底执行定时器                  | checkTimers           |
| 调整 GOMAXPROCS    | 容器 CPU quota 变化时动态调整           | osMaxProcs            |
