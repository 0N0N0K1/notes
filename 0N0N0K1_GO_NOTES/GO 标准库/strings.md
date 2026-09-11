# 1. 作用
实现了用于==操作 UTF-8 编码字符串==的简单函数。

# 2. 开放函数
#### 2.1 查找与包含

| 函数                                           | 说明                 | 示例                                    |
| :------------------------------------------- | :----------------- | :------------------------------------ |
| `Contains(s, substr string) bool`            | 是否包含子串             | `Contains("hello", "ll")` → `true`    |
| `ContainsRune(s string, r rune) bool`        | 是否包含某个 Unicode 字符  | `ContainsRune("hello", 'l')` → `true` |
| `ContainsAny(s, chars string) bool`          | 是否包含 chars 中任意一个字符 | `ContainsAny("hello", "ae")` → `true` |
| `Count(s, substr string) int`                | 子串出现次数             | `Count("banana", "na")` → `2`         |
| `Index(s, substr string) int`                | 子串首次出现位置，无则 `-1`   | `Index("hello", "ll")` → `2`          |
| `LastIndex(s, substr string) int`            | 子串最后一次出现位置         | `LastIndex("banana", "na")` → `4`     |
| `IndexByte(s string, c byte) int`            | 字节首次出现位置           | `IndexByte("hello", 'l')` → `2`       |
| `IndexRune(s string, r rune) int`            | Unicode 字符首次出现位置   | `IndexRune("你好", '好')` → `3`          |
| `IndexFunc(s string, f func(rune) bool) int` | 首个满足条件的字符位置        | —                                     |


#### 2.2 分割与连接

| 函数                                                 | 说明                   |
| :------------------------------------------------- | :------------------- |
| `Split(s, sep string) []string`                    | 按 sep 分割，**sep 被丢弃** |
| `SplitN(s, sep string, n int) []string`            | 最多分割成 n 个子串          |
| `SplitAfter(s, sep string) []string`               | 按 sep 分割，**保留 sep**  |
| `SplitAfterN(s, sep string, n int) []string`       | 同上，限制 n 个            |
| `Fields(s string) []string`                        | 按空白字符（空格、换行、Tab）分割   |
| `FieldsFunc(s string, f func(rune) bool) []string` | 按自定义条件分割             |
| `Join(elems []string, sep string) string`          | 用 sep 连接字符串切片        |

```go
// Split vs SplitAfter
strings.Split("a,b,c", ",")       // ["a", "b", "c"]
strings.SplitAfter("a,b,c", ",")  // ["a,", "b,", "c"]
// Fields 自动处理多余空白
strings.Fields("  a   b  c  ")    // ["a", "b", "c"]
// Join
strings.Join([]string{"a", "b"}, "-")  // "a-b"
```

>  `Split` 的边界情况：`strings.Split("a,b,", ",")` 返回 `["a", "b", ""]`，末尾空串会被保留。

#### 2.3 替换

| 函数                                          | 说明                    |
| :------------------------------------------ | :-------------------- |
| `Replace(s, old, new string, n int) string` | 替换前 n 个，`n < 0` 则替换全部 |
| `ReplaceAll(s, old, new string) string`     | 替换全部（Go 1.12+）        |

```go
strings.Replace("oink oink oink", "oink", "moo", 2)  // "moo moo oink"
strings.ReplaceAll("foo bar foo", "foo", "baz")      // "baz bar baz"
```


#### 2.4 前后缀与裁剪

| 函数                                             | 说明               |
| :--------------------------------------------- | :--------------- |
| `HasPrefix(s, prefix string) bool`             | 是否有指定前缀          |
| `HasSuffix(s, suffix string) bool`             | 是否有指定后缀          |
| `Trim(s, cutset string) string`                | 去掉两端 cutset 中的字符 |
| `TrimLeft(s, cutset string) string`            | 去掉左端             |
| `TrimRight(s, cutset string) string`           | 去掉右端             |
| `TrimSpace(s string) string`                   | 去掉两端空白字符         |
| `TrimPrefix(s, prefix string) string`          | 去掉前缀（只去一次）       |
| `TrimSuffix(s, suffix string) string`          | 去掉后缀（只去一次）       |
| `TrimFunc(s string, f func(rune) bool) string` | 按函数条件裁剪          |

```go
// Trim vs TrimPrefix 的区别
strings.Trim("xxxhelloxxx", "x")      // "hello"      （去掉所有 x）
strings.TrimPrefix("xxxhello", "xx")  // "xhello"     （只去掉开头匹配的 "xx"）

// 常用组合
strings.TrimSpace("  hello  \n")      // "hello"
strings.Trim("...hello...", ".")      // "hello"
```

---

#### 2.5 大小写转换

| 函数                         | 说明                         |
| :------------------------- | :------------------------- |
| `ToUpper(s string) string` | 转大写                        |
| `ToLower(s string) string` | 转小写                        |
| `Title(s string) string`   | 每个单词首字母大写（已废弃，用 `cases` 包） |
| `ToTitle(s string) string` | 转为 Title Case（特殊 Unicode）  |

```go
strings.ToUpper("hello")   // "HELLO"
strings.ToLower("Hello")   // "hello"
```

---

#### 2.6 重复与填充

|函数|说明|
|:--|:--|
|`Repeat(s string, count int) string`|重复 count 次|
|`PadLeft/PadRight`|❌ 标准库没有，需自己实现或用 `fmt.Sprintf`|

```go
strings.Repeat("na", 3)  // "nanana"
```

#### 2.7 比较

|函数|说明|
|:--|:--|
|`Compare(a, b string) int`|字典序比较：`a < b` 返回 `-1`，`a == b` 返回 `0`，`a > b` 返回 `1`|


```go
strings.Compare("a", "b")  // -1
strings.Compare("b", "a")  // 1
strings.Compare("a", "a")  // 0
```

> 实际中直接用 `==`、`<`、`>` 更常见，`Compare` 主要用于排序场景。

---

#### 2.8 strings.Builder（高效拼接）

Go 字符串是**不可变的**，频繁用 `+` 拼接会产生大量临时对象。

```go
var b strings.Builder

// 预分配容量（可选，减少内存分配）
b.Grow(100)

b.WriteString("hello")
b.WriteString(" ")
b.WriteString("world")
b.WriteByte('!')
b.WriteRune('好')

result := b.String()  // "hello world!好"
```

| 方法                                   | 说明              |
| :----------------------------------- | :-------------- |
| `WriteString(s string) (int, error)` | 写入字符串           |
| `WriteByte(c byte) error`            | 写入字节            |
| `WriteRune(r rune) (int, error)`     | 写入 Unicode 字符   |
| `Write(p []byte) (int, error)`       | 实现 io.Writer 接口 |
| `Grow(n int)`                        | 预分配容量           |
| `Len() int`                          | 已写入长度           |
| `Reset()`                            | 重置复用            |
| `String() string`                    | 获取结果            |

> `Builder` 的 `String()` 不会复制底层数据，只是共享切片，所以效率极高。

---

#### 2.9 strings.Reader（字符串作为 Reader）

把字符串包装成 `io.Reader`，方便接入需要 `io.Reader` 的 API。

```go
r := strings.NewReader("hello world")

// 实现了 io.Reader, io.ReaderAt, io.Seeker, io.WriterTo 等接口
buf := make([]byte, 5)
r.Read(buf)  // buf = "hello"
```

---

# 3. 拼接方式

表格

| 方式                | 场景           | 性能             |
| :---------------- | :----------- | :------------- |
| `+`               | 少量、简单的拼接     | 差（每次创建新字符串）    |
| `fmt.Sprintf`     | 需要格式化        | 一般             |
| `strings.Builder` | 大量、循环内拼接     | **最好**         |
| `bytes.Buffer`    | 需要同时处理字节和字符串 | 好              |
| `[]byte` 预分配      | 极致性能，但代码较底层  | 最好（略胜 Builder） |
