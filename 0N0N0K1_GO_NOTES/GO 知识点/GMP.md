
屏蔽线程，并发以协程为粒度，实现M:N调度

| 特性       | OS 线程（1:1 模型） | GMP 模型（M:N 模型）    |
| :------- | :------------ | :---------------- |
| **内存占用** | 栈通常  1~8 MB   | 初始 2KB，动态增长       |
| **创建成本** | 微秒级，需内核参与     | 纳秒级，用户态完成         |
| **切换成本** | 微秒级，需保存大量寄存器  | 纳秒级，仅保存少量上下文      |
| **调度主体** | 操作系统内核        | Go 运行时（用户态）       |
| **阻塞影响** | 阻塞整个线程        | 仅阻塞 G，M 和 P 可分离   |
| **并发规模** | 通常数千线程即达上限    | 轻松支持百万级 Goroutine |
![[Pasted image 20260914103721.png]]

# 1. 核心概念
## 1.1 G — Goroutine（协程）

|           |                  说明                  |
| :-------: | :----------------------------------: |
|  **本质**   |   用户态轻量级执行单元，go func 开辟，由 Go 运行时管理   |
| **栈空间大小** | 拥有自己的栈空间，初始仅 **2KB**，可动态扩缩容（最大约 1GB） |
| **切换成本**  |             纳秒级，无需陷入内核态              |
```go
type g struct {
stack	stack	//实际栈内存区间 [lo, hi)，Go 栈从高位向低位增长
sched	gobuf	//调度快照，保存 sp（栈指针）、pc（程序计数器）、g、bp 等。G 切换时靠它恢复现场
m	*m	//当前 G 运行在哪个 M 上，G→M 的绑定指针
atomicstatus	atomic.Uint32	//G 的生命周期状态：_Gidle、_Grunnable、_Grunning、_Gsyscall、_Gwaiting、_Gdead
schedlink	guintptr	//调度链表指针。G 在全局队列或 P 的本地队列中时，通过它链到下一个 G
goid	uint64	//唯一标识，从 1 开始递增（0 留给 g0）
lockedm	muintptr	//若用户调用了 runtime.LockOSThread()，该 G 被锁定到这个 M 上
waitreason	waitReason	//阻塞原因枚举（如 waitReasonChanSend、waitReasonIOWait）
waiting	*sudog	//G 正在等待的 sudog 链表（channel、select、锁等阻塞场景）
param	unsafe.Pointer	//通用传参通道：channel 唤醒时指向完成的 sudog；GC assist 时传递完成信号
preempt	bool	//收到抢占信号的标志位
preemptStop	bool	//抢占时是直接停止（_Gpreempted）还是仅重新调度

gcAssistBytes	int64	//GC 辅助信用。为正时可无惩罚分配；为负时必须做扫描工作还债
parentGoid / gopc / startpc	uint64/uintptr	//父 G ID、go 语句的 PC、G 入口函数
}
```


## 1.2 M — Machine（线程的封装）

|        |               说明                |
| :----: | :-----------------------------: |
| **本质** |          对操作系统内核线程的封装           |
| **数量** | 默认等于 CPU 核数（`GOMAXPROCS`），可动态增长 |
| **成本** |      创建/切换需陷入内核，成本较高（微秒级）       |

```go
type m struct{
g0	*g	//调度 goroutine。M 执行调度代码时使用的G，与用户 G 隔离
curg	*g	//当前正在运行的用户 G。g0 和 curg 的切换就是「进入/退出调度」
p	puintptr	//当前绑定的 P。M→P 的绑定指针。非空表示拥有该 P 的执行权
nextp	puintptr	//预绑定 P。M 启动或从系统调用返回前，先在这里放好 P，原子交换后正式接管
oldp	puintptr	//进入系统调用前绑定的 P，返回时尝试恢复
spinning	bool	//自旋标志。M 没有工作，正在积极从其他 P 窃取任务。自旋 M 的数量受限制（≤ GOMAXPROCS），避免浪费 CPU
blocked	bool	//M 阻塞在 note 上（无任务可执行时的休眠状态）
park	note	//M 的休眠/唤醒原语，底层用操作系统条件变量或 futex

gsignal	*g	//信号处理 G，拥有独立的信号栈 goSigStack
lockedg	guintptr	//与 g.lockedm 配对，M 被某个 G 锁定（LockOSThread）

alllink	*m	//所有存活 M 的链表（allm），用于遍历和统计
schedlink	muintptr	//空闲 M 链表（sched.freem），M 退出时挂到这里复用
idleNode	listNodeManual	//M 的空闲链表节点~
}
```
## 1.3 P — Processor（逻辑处理器）

|          |                          说明                          |
| :------: | :--------------------------------------------------: |
|  **本质**  |             抽象的「执行上下文」和「本地资源池」，G视角下的CPU              |
|  **数量**  | 由 runtime.GOMAXPROCS(n)决定，默认等于 CPU 核数, **并行数由 P 决定** |
| **核心职责** |                  维护本地 Goroutine 队列                   |

```go
type p struct{
id	int32	//P 的唯一 ID，从 0 到 GOMAXPROCS-1
status	uint32	//状态：Pidle（空闲）、Prunning（运行中）、Psyscall（系统调用中）、Pgcstop（GC 停止）
m	muintptr	//当前绑定的 M。P→M 的绑定指针
oldm	mWeakPointer	//之前运行过的 M（弱引用，M 可能已退出）

runqhead	uint32	//本地队列头指针
runqtail	uint32	//本地队列尾指针
runq	[256]guintptr	//本地可运行 G 队列，循环数组，容量 256
runnext	guintptr	//插队 G。当前 G 通过 go 创建的 G 会放在这里，优先于 runq 执行，减少调度延迟

mcache	*mcache	//内存分配缓存。每个 P 独立持有，小对象分配无锁，避免全局 mcentral 竞争
goidcache / goidcacheend	uint64	//goroutine ID 缓存池，批量从全局 sched.goidgen 申请，减少原子操作
mspancache	struct{len int; buf [128]*mspan}	//mspan 缓存，加速堆内存管理

schedtick	uint32	//每次调用 schedule() 时递增，用于判断是否需要从全局队列取任务（每 61 次检查一次）
syscalltick	uint32	//每次该 P 上的 G 进入/退出系统调用时递增，sysmon 用它检测 M 是否长时间阻塞
}
```
## 1.4 P的本地队列

P的私有G队列，通常由P访问，无锁 , CAS操作获取G
队列操作规则：
1. 生产者（当前运行的 G 创建新 G）：从 runqhead 方向放入
2. 消费者（本地 M 取任务）：从 runqhead 取出
3. 窃取者（其他 M 来偷任务）：从 runqtail 方向偷取一半

## 1.5 全局G队列

是全局调度模块 schedt 的全局共享G队列，存取都需要加锁

# 2. GMP 原理
## 2.1 用户 G 的创建
0. main()函数由全局唯一的M0执行

1. go func() 

编译器会把 go func 翻译成对 runtime.newproc 的调用
参数打包：编译器会把函数 f 和参数 序列化到当前 G 的栈顶，newproc 负责把这些数据拷贝到新 G 的栈

2. runtime.newproc：

|                | 说明                                                  |
| :------------- | :-------------------------------------------------- |
| **G 复用**       | 优先从 `p.gFree` 拿死亡 G，避免频繁 `malloc`                   |
| **goid 缓存**    | 优先用 `p.goidcache`，耗尽后批量从 `sched.goidgen` 申请一批       |
| **初始栈**        | 仅 **2KB**（`_StackMin`）                              |
| **参数拷贝**       | `go` 语句的参数是**值拷贝**到新 G 的栈，与父 G 解耦                   |
| **goexit 兜底**  | `newg.sched.pc` 先设为 `runtime.goexit`，确保 G 函数返回后正确退出 |
| **runnext 优先** | 新 G 优先放入 `runnext`（插队），下次调度直接执行，降低启动延迟              |
3. G 后续状态转换
![[Pasted image 20260914112558.png]]


## 2.2 调度策略
G0 与 G 切换：
1. mcall( ) 从 G —> G0
2. gogo( ) 从 G0 —>G
```plain
优先级从高到低：
1. runnext
   └─ 当前 G 刚 go 出来的 G，延迟最低

2. 每 61 次调度 → 全局队列
   └─ 防止全局队列饥饿（schedtick % 61 == 0）

3. 本地队列

4. 全局队列
   └─ 需要锁，批量取 n 个

5. netpoll（网络 I/O 就绪列表）
   └─ 非阻塞轮询，把就绪 G 批量注入

6. Work Stealing
   └─ 随机选目标 P，从其 本地队列 尾部偷一半，实现负载均衡

7. 自旋（spinning）或休眠（stopm）
```

## 2.3 调度机制

Go 调度器不是基于时间片轮转的，而是==事件驱动 + 协作式 + 抢占式==的混合模型。
### 2.3.1 主动让渡  

指 Goroutine 主动CPU 执行权，把运行机会交给其他 G。

| 触发源        | 入口                         | 原因                  | 操作                                                                                                                     |
| ---------- | -------------------------- | ------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| **用户代码**   | runtime.Gosched() → mcall  | 用户显式让出 CPU，给其他 G 机会 | ① 保存 PC/SP 到 g.sched<br>② 状态 _Grunning → _Grunnable<br>③ dropg() 解除 M 绑定<br>④ globrunqput() 放入全局队列<br>⑤ schedule()找新 G |
| **G 正常退出** | goexit1() → mcall(goexit0) | 用户函数 return         | ① 状态 _Grunning → _Gdead<br>② 释放 defer、panic 链<br>③ gfput() 放入 p.gFree 复用<br>④ schedule()找新 G                           |
### 2.3.2 协作式让渡

设置 stackguard0 = stackPreempt，请求当前 G 在==下一次函数调用时让渡==：

| 触发源                  | 入口                                                                                                           | 原因                 | 操作                                                                                                                                                      |
| -------------------- | ------------------------------------------------------------------------------------------------------------ | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **[[sysmon ]]时间片检测** | sysmon() → retake() → preemptone() 设置 stackguard0 = stackPreempt → 下次函数调用触发 morestack() → mcall(largerstack) | G 连续运行超过 **10ms**  | ① 检查 stackguard0 == stackPreempt<br>② 状态 _Grunning → _Grunnable<br>③ stackguard0 重置为正常值<br>④ runqput(pp, gp, true) 放入**本地 runnext**<br>⑤ schedule()找新 G |
| **GC STW**           | stopTheWorld() → preemptStop = true + preemptone() → morestack() → mcall(preemptPark)                        | GC 需要全局暂停做标记       | ① 状态 _Grunning → _Gpreempted<br>② dropg()挂到 STW 等待列表<br>③ 阻塞直到 GC 结束被唤醒                                                                                 |
| **GC 标记辅助**          | gcAssistAlloc() 分配时发现 gcAssistBytes < 0 → 还债后可能 gopark                                                       | 分配速度超过 GC 标记速度，需还债 | ① 执行 gcDrain() 帮 GC 扫描<br>② 债务还清后返回，或让渡                                                                                                                 |
| **栈收缩**              | preemptShrink = true → 函数调用触发 morestack() → shrinkstack()                                                    | 栈使用率低，回收内存         | ① 拷贝栈到更小内存块<br>② 继续执行原 G                                                                                                                                |
| **栈扩容**              | morestack() → mcall(newstack)                                                                                | 栈空间不足（真实溢出，非抢占）    | ① 分配 2 倍新栈<br>② 拷贝旧栈数据<br>③ 调整指针<br>④ 切回继续执行                                                                                                            |

### 2.3.3 抢占式让渡

**问题**：协作式让渡依赖**函数调用**作为检查点。纯计算循环 G（无函数调用）没有安全点，即使运行几个小时也不会让出 CPU。

抢占式让渡：SIGURG 信号强制打断当前指令，不管 G 愿不愿意
生效时机：信号到达后 CPU 立刻暂停

| 触发源              | 入口                                                                                                                                 | 原因                               | 操作                                                                                                                                                         |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------- | -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **[[sysmon]]检测** | sysmon() → retake() → preemptone() → signalM(mp, SIGURG) → sighandler() 改 PC → asyncPreempt → asyncPreempt2() → mcall(gopreempt_m) | G 运行超过 **10ms** 且函数调用检查无效（纯计算循环） | ① pthreadkill 发 SIGURG<br>② stackguard0 = stackPreempt<br>③ 信号返回后执行 asyncPreempt（汇编）<br>④ **保存全部寄存器**<br>⑤ asyncPreempt2 检查标志<br>⑥ mcall(gopreempt_m) 重新调度 |
| **GC STW（信号版）**  | stopTheWorld() → preemptStop = true → signalM(SIGURG) → asyncPreempt2() → mcall(preemptPark)                                       | GC 要求所有 G **立即停止**               | ① 信号强制打断<br>② asyncPreempt2 发现 preemptStop=true<br>③ mcall(preemptPark) 挂起<br>④ 状态 _Gpreempted，等 GC 结束                                                     |
| **GC 标记辅助（信号版）** | `gcAssistAlloc()` 时 debt 过大 → 设置抢占标志 → 信号触发                                                                                        | 分配太快，强制帮 GC 干活                   | ① 抢占触发<br>② 执行标记辅助工作<br>③ 债务清零后继续运行                                                                                                                        |
### 2.3.4 阻塞调度

![[Pasted image 20260914132938.png]]

| 触发源                   | 入口                                               | 原因                   | 操作                                                                                                   |
| --------------------- | ------------------------------------------------ | -------------------- | ---------------------------------------------------------------------------------------------------- |
| **channel 接收/发送阻塞**   | chansend() → gopark(chanparkcommit, ...)         | 无缓冲/无对端              | ① 封装 sudog 挂入 channel 发送等待队列<br>② park_m() 状态 → _Gwaiting<br>③ handoffp() P 移交其他 M<br>④ stopm() M 休眠 |
| **select 阻塞**         | selectgo() → gopark(selparkcommit, ...)          | 所有 case 都无法执行        | ① 所有 case 的 sudog 都挂入对应 channel<br>② gopark 挂起<br>③ 任一 case 就绪时 goready 唤醒                           |
| **sync.Mutex 阻塞**     | lock2() → goparkunlock(&l.key, ...)              | 锁被其他 G 持有            | ① 挂入 mutex 等待队列<br>② goparkunlock（先解锁调度器再 park）<br>③ Unlock 时 goready 唤醒队首                           |
| **sync.WaitGroup 等待** | Wait() → gopark()                                | counter > 0，需等待 Done | ① 记录等待者<br>② gopark 挂起<br>③ Done 归零时唤醒所有等待者                                                          |
| **time.Sleep**        | time.Sleep() → gopark() + timer                  | 定时休眠                 | ① 注册 timer（g.timer）<br>② gopark 挂起<br>③ 定时器到期 → goready 唤醒                                           |
| **网络 I/O 未就绪**        | netpollblock() → gopark(netpollblockcommit, ...) | socket 数据未就绪         | ① 挂入 pollDesc 等待队列<br>② **M 不阻塞**，继续 schedule<br>③ sysmon/netpoll 检测到就绪 → goready 唤醒                 |
### 2.3.5 系统调用调度

 系统调用：M 即将陷入内核态（磁盘I/O，cgo 调用）
![[Pasted image 20260914125328.png]]

### 2.3.6 M 的休眠与唤醒

 当 M 找不到任务时，不是立即休眠，而是**自旋一段时间**：
 
-  **自旋 M 数量限制**：2 * nmspinning < GOMAXPROCS - npidle
- 自旋时 M 在**忙等**（循环检查队列），消耗 CPU
- 但自旋 M 可以**立即响应**新任务，无需操作系统上下文切换

唤醒时先检查有无自旋，再尝试唤醒，最后用原子 CAS 保证只有一个在尝试新建 M
