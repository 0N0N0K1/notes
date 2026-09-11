# 1. 作用
与操作系统交互的核心包，提供了==文件系统操作、进程控制、环境变量访问==等底层能力。几乎所有 Go 程序都会直接或间接用到它。
## 2. 文件与目录操作
### 2.1. 文件操作
#### 打开文件
```go
// 只读打开
f, err := os.Open("data.txt")

// 读写打开，不存在则创建，存在则清空
f, err := os.Create("output.txt")

// 精细控制：指定打开模式、权限
f, err := os.OpenFile("log.txt", os.O_APPEND|os.O_WRONLY|os.O_CREATE, 0644)
```

**打开模式标志**（可组合使用 `|`）：

|常量|含义|
|:--|:--|
|`os.O_RDONLY`|只读|
|`os.O_WRONLY`|只写|
|`os.O_RDWR`|读写|
|`os.O_APPEND`|追加写入|
|`os.O_CREATE`|不存在则创建|
|`os.O_TRUNC`|存在则清空|
|`os.O_EXCL`|与 O_CREATE 配合，文件已存在则报错|
|`os.O_SYNC`|同步 I/O|

**文件权限**（Unix 风格，八进制）：
- `0644`：所有者读写，其他人只读
- `0755`：所有者读写执行，其他人读执行
- `0600`：仅所有者可读写
#### 读写关闭
```go
// 实现了 io.Reader / io.Writer / io.Closer
data := make([]byte, 1024)
n, err := f.Read(data)
n, err := f.Write([]byte("hello"))

// 定位
offset, err := f.Seek(0, io.SeekStart)  // 回到开头
offset, err := f.Seek(0, io.SeekEnd)    // 跳到末尾
offset, err := f.Seek(100, io.SeekCurrent) // 从当前位置偏移100

// 截断文件到指定大小
err := f.Truncate(100)
// 同步到磁盘
err := f.Sync()  // 类似 fsync，确保数据落盘
// 关闭（重要！）
err := f.Close()
```

#### 便捷一次性读写
```go
// 读取整个文件
data, err := os.ReadFile("config.json")

// 写入整个文件（覆盖）
err := os.WriteFile("output.txt", []byte("data"), 0644)
```

#### 文件信息
```go
info, err := f.Stat()  // 或 os.Stat("path")
info.Name()    // 文件名
info.Size()    // 字节大小
info.Mode()    // 权限模式
info.ModTime() // 修改时间
info.IsDir()   // 是否是目录
fd := f.Fd()  // 获取底层文件描述符（uintptr）
```

### 2.2. 目录操作
```go
// 创建目录
err := os.Mkdir("mydir", 0755)
// 递归创建（类似 mkdir -p）
err := os.MkdirAll("a/b/c", 0755)
// 删除目录（必须为空）
err := os.Remove("mydir")
// 递归删除（类似 rm -rf）
err := os.RemoveAll("mydir")
// 读取目录内容
entries, err := os.ReadDir(".")  // Go 1.16+，返回 []os.DirEntry
for _, entry := range entries {
    fmt.Println(entry.Name(), entry.IsDir())
}
// 旧版（Go 1.16 之前）
names, err := os.ReadDir(".")  // 返回文件名列表
```

### 2.3. 文件系统元信息
```go
// 获取文件信息
info, err := os.Stat("file.txt")
// 检查文件是否存在（利用 Stat 的错误）
if _, err := os.Stat("file.txt"); os.IsNotExist(err) {
    // 文件不存在
}
// 检查是否是某种错误
os.IsExist(err)      // 是否已存在
os.IsPermission(err) // 是否权限不足
os.IsNotExist(err)   // 是否不存在
// 修改权限
err := os.Chmod("file.txt", 0755)
// 修改所有者（Unix）
err := os.Chown("file.txt", uid, gid)
// 重命名/移动
err := os.Rename("old.txt", "new.txt")
// 创建硬链接
err := os.Link("a.txt", "b.txt")
// 创建符号链接
err := os.Symlink("a.txt", "link.txt")
// 读取符号链接指向的真实路径
target, err := os.Readlink("link.txt")
```

### 4. 临时文件/目录
```go
// 创建临时文件（在系统临时目录中）
f, err := os.CreateTemp("", "prefix-*.txt")
defer os.Remove(f.Name())  // 用完记得删

// 创建临时目录
dir, err := os.MkdirTemp("", "prefix-*")
defer os.RemoveAll(dir)
```
# 3. 标准输入输出与进程

### 3.1. 标准 IO
```go
os.Stdin   // *os.File，标准输入
os.Stdout  // *os.File，标准输出
os.Stderr  // *os.File，标准错误
```

### 3.2. 进程信息
```go
os.Getpid()       // 当前进程 ID
os.Getppid()      // 父进程 ID
os.Getuid()       // 当前用户 ID（Unix）
os.Getgid()       // 当前组 ID（Unix）
os.Getwd()        // 当前工作目录
os.Getenv("PATH") // 获取环境变量
os.Hostname()     // 主机名
```

### 3. 3环境变量
```go
// 获取
path := os.Getenv("PATH")
// 设置
os.Setenv("MY_VAR", "value")
// 删除
os.Unsetenv("MY_VAR")
// 获取全部
for _, e := range os.Environ() {
    // "KEY=VALUE"
}
```

### 3.4. 退出与信号
```go
os.Exit(1)        // 立即退出程序（不会执行 defer！）
os.Exit(0)        // 正常退出

// 更优雅的退出通常用：
// import "os/signal"
```