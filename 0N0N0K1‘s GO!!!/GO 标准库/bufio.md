# 1. 作用

直接使用 os.File 或 net.Conn 进行读写时，每次 Read/Write 都会触发一次系统调用。系统调用涉及用户态/内核态切换，开销较大，bufio的核心作用是在 [[io]].Reader/[[io]].Writer 之上加一层内存缓冲进行封装，==包装已有的 io.Reader / io.Writer==（文件、网络连接等），通过缓冲优化 I/O，==大幅减少系统调用次数，从而显著提升 I/O 性能==

# 2. 核心类型
- 简单按行读文本 → 用 `Scanner`
- 需要 `Peek`、处理大文件、自定义协议 → 用 `Reader`
#### 2.1. bufio.Reader
```go
//为 io.Reader 对象实现了缓冲机制。可以通过调用 [NewReader] 或 [NewReaderSize] 创建一个新的 Reader 对象
type Reader struct {  
    buf          []byte  
    rd           io.Reader // reader provided by the client  
    r, w         int       // buf read and write positions  
    err          error  
    lastByte     int // last byte read for UnreadByte; -1 means invalid  
    lastRuneSize int // size of last rune read for UnreadRune; -1 means invalid  
}

//封装io.Reader返回bufio.Reader实例
func NewReader(rd io.Reader) *Reader               // 默认缓冲区 4KB
func NewReaderSize(rd io.Reader, size int) *Reader // 自定义缓冲区大小


```   

| 方法                                                                  | 说明                        |
| ------------------------------------------------------------------- | ------------------------- |
| func (b *Reader) Read(p []byte) (n int, err error)                  | 实现 `io.Reader`，优先从缓冲区读    |
| func (b *Reader) ReadByte() (byte, error)                           | 读一个字节                     |
| func (b *Reader) ReadBytes(delim byte) ([]byte, error)              | 读到指定分隔符，返回 `[]byte`       |
| func (b *Reader) ReadString(delim byte) (string, error)<br>         | 读到指定分隔符，返回 `string`       |
| func (b *Reader) ReadLine() (line []byte, isPrefix bool, err error) | 读一行（不推荐，有长度限制）            |
| func (b *Reader) Peek(n int) ([]byte, error)                        | 预读前 n 个字节（不移动读指针）         |
| func (b *Reader) Discard(n int) (discarded int, err error)          | 跳过 n 个字节                  |
| func (b *Reader) Reset(r io.Reader)                                 | 重置底层 Reader，复用缓冲区         |
| func (b *Reader) ReadRune() (r rune, size int, err error)           | 读取一个 UTF-8 编码的 Unicode 字符 |
| func (*Reader) Buffered                                             | 返回已使用缓冲区大小                |
| func (b *Reader) WriteTo(w io.Writer) (n int64, err error)<br>      | 向writer中写入                |
#### 2.2. bufio.Writer
```go
//包装 `io.Writer`，提供缓冲写入能力
type Writer struct {  
    err error  
    buf []byte  
    n   int  
    wr  io.Writer  
}
// //封装io.Writer返回bufio.Writer实例
func NewWriter(w io.Writer) *Writer
func NewWriterSize(w io.Writer, size int) *Writer

```

| 方法                                                              | 说明                                                                                        |
| --------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| func (b *Writer) Write(p []byte) (nn int, err error)<br>        | 实现 `io.Writer`，先写入缓冲区                                                                     |
| func (b *Writer) WriteByte(c byte) error                        | 写一个字节                                                                                     |
| func (b *Writer) WriteString(s string) (int, error)<br>         | 写字符串（避免转 `[]byte`）                                                                        |
| ==func (b *Writer) Flush() error==<br>                          | ==**刷新缓冲区到底层，bufio.Writer的数据不会立即写入底层，必须调用 Flush(）**==             **        defer Flush** |
| func (b *Writer) Available() int<br>                            | 返回缓冲区剩余可用空间                                                                               |
| func (b *Writer) Buffered() int                                 | 返回当前缓冲区中已写入未刷新的字节数                                                                        |
| func (b *Writer) Reset(w io.Writer)                             | 重置底层 Writer                                                                               |
| func (b *Writer) Size() int                                     | 返回buf大小                                                                                   |
| func (b *Writer) ReadFrom(r io.Reader) (n int64, err error)<br> | 从Reader中读                                                                                 |
#### 2.3. bufio.Scanner

```go
//用于逐 token 读取数据，是 `bufio.Reader` 的更高层封装，使用更简单
type Scanner struct {  
    r            io.Reader // The reader provided by the client.  
    split        SplitFunc // The function to split the tokens.    
    maxTokenSize int       // Maximum size of a token; modified by tests.  
    token        []byte    // Last token returned by split.  
    buf          []byte    // Buffer used as argument to split.  
    start        int       // First non-processed byte in buf.  
    end          int       // End of data in buf.  
    err          error     // Sticky error.  
    empties      int       // Count of successive empty tokens.  
    scanCalled   bool      // Scan has been called; buffer is in use.    
    done         bool      // Scan has finished.
    }
    
    func NewScanner(r io.Reader) *Scanner  //64KB 限制
    
    //缓冲区控制扫描器分配内存,可通过 `scanner.Buffer()` 扩大：
    func (s *Scanner) Buffer(buf []byte, max int)


```

```GO

func (s *Scanner) Split(split SplitFunc)
type SplitFunc func(data []byte, atEOF bool) (advance int, token []byte, err error)

// 按行（默认）
scanner.Split(bufio.ScanLines)
// 按单词（以空白字符分隔）
scanner.Split(bufio.ScanWords)
// 按字节
scanner.Split(bufio.ScanBytes)
// 自定义：按逗号分割
scanner.Split(func(data []byte, atEOF bool) (advance int, token []byte, err error) {
    if atEOF && len(data) == 0 {
        return 0, nil, nil
    }
    if i := bytes.IndexByte(data, ','); i >= 0 {
        return i + 1, data[:i], nil
    }
    if atEOF {
        return len(data), data, nil
    }
    return 0, nil, nil
})
```
#### 2.4. bufio.ReadWriter
```go
type ReadWriter struct {
	*Reader
	*Writer
}
```

# 3. 最佳实践
1. 大文件传输
```go
func CopyFile(src, dst string) error {
    in, err := os.Open(src)
    if err != nil {
        return err
    }
    defer in.Close()

    out, err := os.Create(dst)
    if err != nil {
        return err
    }
    defer out.Close()

    // 双方都用缓冲，性能最佳
    reader := bufio.NewReader(in)
    writer := bufio.NewWriter(out)
    defer writer.Flush()

    _, err = io.Copy(writer, reader)
    return err
}
```
2. HTTP收发
```go
func handleConn(conn net.Conn) {
    defer conn.Close()
    
    reader := bufio.NewReader(conn)
    writer := bufio.NewWriter(conn)
    
    // 读取请求行
    requestLine, err := reader.ReadString('\n')
    if err != nil {
        return
    }
    
    // 写入响应
    writer.WriteString("HTTP/1.1 200 OK\r\n")
    writer.WriteString("Content-Length: 5\r\n")
    writer.WriteString("\r\n")
    writer.WriteString("hello")
    
    writer.Flush()  // 必须刷新！
}
```
3. 资源释放
```go
//bufio.Reader/Writer 本身不持有需要关闭的资源，但底层 Reader/Writer 需要关闭

f, _ := os.Open("file")
defer f.Close()  // 关文件，不是关 bufio.Reader
reader := bufio.NewReader(f)
```