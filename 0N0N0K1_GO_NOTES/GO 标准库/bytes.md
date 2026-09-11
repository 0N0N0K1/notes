# 1. 作用
用于==操作字节切片（[]byte）的工具集合==。直接==使用内存中的字节切片，不涉及底层 I/O==,它提供了与 [[strings]]包高度相似的函数，但操作对象是 []byte 而非字符串。由于 Go 中字符串是不可变的，而字节切片是可变的，因此 bytes 包特别适合需要频繁修改内容的场景，如缓冲区构建、数据解析等

# 2核心类型
#### 2.1bytes.Buffer
Buffer 是一个可变大小的字节缓冲区，实现了 [[io]].Reader、[[io]].Writer、[[io]].ByteReader、[[io]].ByteWriter、[[io]].StringWriter 等多个接口
==是构建和操作字节数据的核心类型==
```go
type Buffer struct {  
    buf      []byte // contents are the bytes buf[off : len(buf)]  
    off      int    // read at &buf[off], write at &buf[len(buf)]  
    lastRead readOp // last read operation, so that Unread* can work correctly.  
  }
```
创建与初始化
```go
var buf bytes.Buffer          // 零值可用，无需初始化
buf := bytes.NewBuffer([]byte("hello"))  // 使用现有切片初始化
buf := bytes.NewBufferString("hello")    // 使用字符串初始化
```
常用方法
```go
//写入
Write(p []byte) (int, error)
WriteString(s string)
WriteByte(c byte)、WriteRune(r rune)

//读取
Read(p []byte) (int, error)
ReadByte() (byte, error)
ReadRune() (rune, int, error)
ReadString(delim byte) (string, error)

//其他
Bytes() []byte//返回缓冲区内容的切片（注意：返回的是底层数据的引用，修改需谨慎）

String() string//返回缓冲区内容的字符串

Len() int//当前未读部分的长度

Cap() int//底层切片的容量

Truncate(n int)//保留前 n 个未读字节

Reset()//清空缓冲区

Grow(n int)//增加缓冲区容量

Next(n int) []byte//读取并返回接下来的 n 个字节，同时移动读指针

ReadFrom(r io.Reader) (int64, error)//从 Reader 读取所有数据并写入缓冲区

WriteTo(w io.Writer) (int64, error)//将缓冲区未读部分写入 Writer
```
#### 2.2 bytes.Reader
Reader 实现了 io.Reader、io.ReaderAt、io.Seeker、io.ByteReader、io.RuneReader 等接口，==用于从字节切片中读取数据，支持随机访问和定位==。
```go
type Reader struct {  
    s        []byte  
    i        int64 // current reading index  
    prevRune int   // index of previous rune; or < 0  
}
```
创建
```go
r := bytes.NewReader([]byte("Hello World"))
```
常用方法
```go
Len() int//未读取部分的长度

Size() int64//底层切片的总长度

Read(p []byte) (int, error)
ReadAt(p []byte, off int64) (int, error)
ReadByte() (byte, error)
ReadRune() (rune, int, error)

Seek(offset int64, whence int) (int64, error)//移动读指针，whence 可为 io.SeekStart、io.SeekCurrent、io.SeekEnd

WriteTo(w io.Writer) (int64, error)//将未读部分写入 Writer
```


# 3.  常用函数
bytes 包提供了大量与 [strings] 类似的函数，以下分类介绍。

 #### 3.1 比较与相等
```go
Compare(a, b []byte) int//按字典序比较，返回 -1、0、1

Equal(a, b []byte) bool//判断两个切片内容是否完全相同

EqualFold(s, t []byte) bool//忽略大小写比较（基于 Unicode 简单折叠）

Contains(b, subslice []byte) bool//判断是否包含子切片

ContainsAny(b []byte, chars string) bool//判断是否包含 chars 中任意一个字符

ContainsRune(b []byte, r rune) bool//判断是否包含指定 rune

HasPrefix(s, prefix []byte) bool//是否以 prefix 开头

HasSuffix(s, suffix []byte) bool//是否以 suffix 结尾
```
#### 3.2 索引与查找
```go
Index(s, sep []byte) int//子切片第一次出现的位置，不存在返回 -1

IndexAny(s []byte, chars string) int//chars 中任一字符出现的位置

IndexByte(s []byte, c byte) int//字符 c 出现的位置

IndexRune(s []byte, r rune) int//rune 出现的位置

IndexFunc(s []byte, f func(r rune) bool) int//第一个满足 f 的 rune 的字节索引

LastIndex(s, sep []byte) int//子切片最后一次出现的位置

LastIndexAny(s []byte, chars string) int

LastIndexByte(s []byte, c byte) int

LastIndexFunc(s []byte, f func(r rune) bool) int

Count(s, sep []byte) int//sep 出现的次数（不重叠）
```
#### 3.3 分割与连接
```go
Split(s, sep []byte) [][]byte//按 sep 分割，返回所有子切片（包括空）

SplitN(s, sep []byte, n int) [][]byte//最多分割 n 次

SplitAfter(s, sep []byte) [][]byte//分割后保留 sep

SplitAfterN(s, sep []byte, n int) [][]byte

Fields(s []byte) [][]byte//按空白字符分割

FieldsFunc(s []byte, f func(rune) bool) [][]byte//按自定义函数分割

Join(s [][]byte, sep []byte) []byte//用 sep 连接切片
```
#### 3.4 转换与修改
```go

ToUpper(s []byte) []byte、ToLower(s []byte) []byte//大小写转换（返回新切片）

ToTitle(s []byte) []byte//转换为标题大小写

Trim(s []byte, cutset string) []byte//去除两端 cutset 中出现的字符

TrimSpace(s []byte) []byte//去除两端空白

TrimPrefix(s, prefix []byte) []byte//去除前缀

TrimSuffix(s, suffix []byte) []byte：//去除后缀

TrimFunc(s []byte, f func(r rune) bool) []byte//去除两端满足条件的 rune

//TrimLeft、TrimRight 等类似

Replace(s, old, new []byte, n int) []byte//替换前 n 个 old 为 new（n<0 表示全部）

ReplaceAll(s, old, new []byte) []byte//换所有

Map(mapping func(r rune) rune, s []byte) []byte//对每个 rune 应用映射函数
```
#### 3.5 其他实用函数
```go
Repeat(b []byte, count int) []byte //重复 count 次

Runes(s []byte) []rune //将字节切片转换为 rune 切片（按 UTF-8 解码）

```


# 4.常见陷阱

1. Buffer.Bytes() 返回底层切片：修改返回的切片会影响 Buffer 内容；同理，对 Buffer 的后续写入可能导致原切片失效（因为底层数组可能重新分配）。若需要独立副本，使用 append([]byte(nil), buf.Bytes()...)。

2. Buffer 的读写指针：Read 方法会移动内部读指针，使用 Bytes() 或 String() 获取的是未读部分。若想获取全部内容而不影响指针，可先 Reset 再获取（不推荐），或使用 Next(buf.Len())。

3. Split 与 SplitN 的边界情况：当 sep 为空时，Split 会按 UTF-8 字符分割，与 strings.Split 行为一致。

4. Trim 的 cutset 是字符集合：不是前缀/后缀字符串，而是包含在 cutset 中的任意字符都会被去除。