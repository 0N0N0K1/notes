# 1. 作用

直接使用 os.File 或 net.Conn 进行读写时，每次 Read/Write 都会触发一次系统调用。系统调用涉及用户态/内核态切换，开销较大，bufio的核心作用是在 io.Reader/io.Writer 之上加一层内存缓冲，==大幅减少系统调用次数，从而显著提升 I/O 性能==

# 2. 核心接口
#### 2.1. Reader