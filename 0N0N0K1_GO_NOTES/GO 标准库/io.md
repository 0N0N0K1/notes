# 1. 作用

io 包的本质是 **接口驱动**，==为其他包实现 I/O 操作提供了统一的接口，需使用者实现接口==。涉及 **文件、网络、缓冲** 等包的操作
[[os]]  ： os.File 实现了 io.ReadWriteCloser
net： net.Conn 实现了 io.ReadWriteCloser
[[bytes]] ：bytes.Buffer 实现 io.Reader/io.Writer
[[strings]] ：strings.Reader 实现 io.Reader/io.ReaderAt
[[bufio]] ：包装 io.Reader/io.Writer 提供缓冲 
errors : 


# 2. 四大核心接口

#### 2.1. io.**Reader**
```go
type Reader interface {
    // n为从实现Read方法对象中读入到p中字节数，err 为返回错误类型（其中err==io.EOF为读到文件结束的返回值）
    Read(p []byte) (n int, err error)
}
```
#### 2.2. io.**Writer**
```go
type Writer interface {
    // n为从p写入到实现Read方法对象中字节数，err 为返回错误类型
    Write(p []byte) (n int, err error)
}
```
#### 2.3. **io.Closer**
```go
type Closer interface {
   // 关闭当前资源/连接
    Close() error
}
```
#### 2.4. io.**Seeker**
```go
type Seeker interface {
    // 用于设置偏移量随机访问
    Seek(offset int64, whence int) (int64, error)
}
```
#### 2.5. **组合接口**

| 接口名               | 组合                             |
| ----------------- | ------------------------------ |
| `ReadWriter`      | `Reader` + `Writer`            |
| `ReadCloser`      | `Reader` + `Closer`            |
| `WriteCloser`     | `Writer` + `Closer`            |
| `ReadWriteCloser` | `Reader` + `Writer` + `Closer` |
| `ReadSeeker`      | `Reader` + `Seeker`            |
| `ReadWriteSeeker` | `Reader` + `Writer` + `Seeker` |
# 3.开放函数
1. io.copy...
```go
// 将数据从 src 复制到 dst，直到 EOF 或出错。
func Copy(dst Writer, src Reader) (written int64, err error)

// 带buf缓冲的复制，它会直接使用提供的缓冲区（如果需要），而不是分配临时缓冲区。如果 buf 为 nil，则会分配一个缓冲区；否则，如果 buf 的长度为零，则 CopyBuffer 会引发 panic。
func CopyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error)

// 指定复制字节数
func CopyN(dst Writer, src Reader, n int64) (written int64, err error)
```
2. io.pipe
```go
//Pipe 创建一个同步的 内存管道。 返回该内存的 io.Reader ，io.Writer 
func Pipe() (*PipeReader, *PipeWriter)
```
3. io.Read...
```go
//从源 r 读取数据，直到遇到错误或文件末尾 (EOF) 为止，并返回读取到的数据
func ReadAll(r Reader) ([]byte, error)

//从 r 读取数据到 buf 缓冲区，直到读取至少 min 个字节为止。它返回已复制的字节数，
//1.如果读取的字节数少于 min，则返回错误。
//2.仅当未读取任何字节时，错误才会返回 EOF。
//3.如果在读取少于 min 个字节后遇到 EOF，ReadAtLeast 函数返回 ErrUnexpectedEOF 。
//4.如果 min 大于 buf 缓冲区的长度，ReadAtLeast 函数返回 ErrShortBuffer 。
func ReadAtLeast(r Reader, buf []byte, min int) (n int, err error)

//从 r 中读取 len(buf) 个字节到 buf 中。
//它返回已复制的字节数，如果读取的字节数少于 len(buf)，则返回错误。
//仅当未读取任何字节时，错误才会返回 EOF。
//如果在读取部分字节后遇到 EOF，ReadFull 函数将返回 ErrUnexpectedEOF
func ReadFull(r Reader, buf []byte) (n int, err error)
```
4. io.writestring
```go
//将字符串 s 的内容写入对象 w
func WriteString(w Writer, s string) (n int, err error)
```
5. io.TeeRead
```go
//TeeReader 返回一个 Reader ，该对象将从 r 读取的内容写入 w。 对 r 执行的读取操作前读取内容先写入 w
func TeeReader(r Reader, w Writer) Reader
```
# 4. 错误类型
```go 
//当没有更多输入可用时，Read 函数会返回 EOF 错误。函数应该仅在表示输入正常结束时才返回 EOF。如果在结构化数据流中意外出现 EOF，则应返回 ErrUnexpectedEOF 或其他提供更多详细信息的错误。
var EOF = errors.New("EOF")

//用于对已关闭的管道进行读取或写入操作时的错误。
var ErrClosedPipe = errors.New("io: read/write on closed pipe")
 
//当 Reader 的某些客户端多次调用未能返回任何数据或错误时，ErrNoProgress 会被返回，这通常是 Reader 实现损坏的标志
var ErrNoProgress = errors.New("multiple Read calls return no data or error")

//表示读取操作需要的缓冲区比提供的缓冲区要大。
var ErrShortBuffer = errors.New("short buffer")

//表示写入操作接收到的字节数少于请求的字节数，但未能返回明确的错误信息。
var ErrShortWrite = errors.New("short write")

//表示在读取固定大小的数据块或数据结构的过程中遇到了 EOF。
var ErrUnexpectedEOF = errors.New("unexpected EOF")
```

# 5. 其他接口
类型过多，具体参考[官方文档](https://pkg.go.dev/io#pkg-types)
```go 
type WriterAt interface {
	WriteAt(p []byte, off int64) (n int, err error)
}

type WriterTo interface {
	WriteTo(w Writer) (n int64, err error)
}

type RuneScanner interface {
	RuneReader
	UnreadRune() error


type RuneReader interface {
	ReadRune() (r rune, size int, err error)
}
.
.
.
.
.
.

```