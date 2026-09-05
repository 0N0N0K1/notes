# 1. 作用
Go 最基础、最常用、必须熟练的包，进行格式化输入输出的核心工具，掌握其常用函数和格式化动词对于编写清晰、可读的 Go 代码至关重要。通过实现 `Stringer` 和 `Formatter` 接口，我们可以让自定义类型具有友好的打印格式。

# 2. 格式化动词（Verbs）—— 核心中的核心

通用

| 动词    | 说明              | 示例                                              |
| :---- | :-------------- | :---------------------------------------------- |
| `%v`  | 默认格式            | 任何类型                                            |
| `%+v` | 结构体时带字段名        | `fmt.Printf("%+v", struct{A int}{1})` → `{A:1}` |
| `%#v` | Go 语法表示（可用来复现值） | `fmt.Printf("%#v", "a")` → `"a"`                |
| `%T`  | 打印类型            | `fmt.Printf("%T", 3.14)` → `float64`            |
| `%%`  | 输出百分号           | `%%` → `%`                                      |

 布尔、整数

| 动词          | 说明           |
| :---------- | :----------- |
| `%t`        | 布尔值          |
| `%d`        | 十进制整数        |
| `%b`        | 二进制          |
| `%o`        | 八进制          |
| `%O`        | 八进制带 `0o` 前缀 |
| `%x` / `%X` | 十六进制（小写/大写）  |

浮点数、复数

| 动词          | 说明               |
| :---------- | :--------------- |
| `%f`        | 小数表示（默认 6 位）     |
| `%.2f`      | 保留 2 位小数         |
| `%e` / `%E` | 科学计数法            |
| `%g` / `%G` | 自动选择 `%f` 或 `%e` |

字符串、字节切片

| 动词          | 说明                      |
| :---------- | :---------------------- |
| `%s`        | 字符串或 `[]byte` 原样输出      |
| `%q`        | 带双引号的 Go 字符串字面量（转义特殊字符） |
| `%x` / `%X` | 每个字节转十六进制               |

指针

|动词|说明|
|:--|:--|
|`%p`|指针地址，十六进制|

 标志

|标志|说明|
|---|---|
|`+`|总是输出正负号|
|`-`|左对齐|
|`#`|备用格式（如 `%#x` 加 `0x` 前缀）|
||正数前留空格（仅数值）|
|`0`|用 0 填充宽度|

```go
fmt.Printf("|%10d|\n", 42)     // |        42|  右对齐，宽度10
fmt.Printf("|%-10d|\n", 42)    // |42        |  左对齐
fmt.Printf("|%010d|\n", 42)    // |0000000042|  前导零
fmt.Printf("|%.2f|\n", 3.1415) // |3.14|       保留2位小数
fmt.Printf("|%10.2f|\n", 3.14) // |      3.14|  宽度10，精度2
```


# 3. 输出系列

向os.Stdout输出（print 系列）

| 函数                                | 说明                |
| :-------------------------------- | :---------------- |
| `Print(a ...any)`                 | 直接输出，参数间无空格，末尾无换行 |
| `Println(a ...any)`               | 输出，参数间有空格，末尾自动换行  |
| `Printf(format string, a ...any)` | 按格式字符串输出          |

向 [[io]].Writer 输出（Fprint 系列）

|函数|说明|
|:--|:--|
|`Fprint(w io.Writer, a ...any)`|向任意 `io.Writer` 输出|
|`Fprintln(...)`|同上，加换行|
|`Fprintf(w io.Writer, format string, a ...any)`|格式化输出到 Writer|


# 4. 格式化字符串（不输出，只返回字符串）

| 函数                                                          | 说明                                                                                                       |
| :---------------------------------------------------------- | :------------------------------------------------------------------------------------------------------- |
| `Sprint(a ...any) string`                                   | 拼接为字符串                                                                                                   |
| `Sprintln(a ...any) string`                                 | 拼接为字符串，参数间有空格，末尾换行                                                                                       |
| `Sprintf(format string, a ...any) string`                   | 按格式返回字符串                                                                                                 |
| `func Errorf(format string, a ...any) (err error`)`<br><br> | 错误格式化                                                      %v	仅嵌入错误信息，不建立链<br>%w	包装错误，建立错误链（可被 Is/As 追溯） |

`Errorf` 根据格式返回一个 `error` 类型的值，常用于创建带有格式化信息的错误。

go



# 5. 输入扫描（Scan 系列）
 
 标准输入系列

|函数|说明|
|---|---|
|`Scan`|从标准输入读取，按空格分隔|
|`Scanf`|根据格式字符串读取|
|`Scanln`|读取一行，按空格分隔|

 从 [[io]].Reader 读取

|函数|说明|
|---|---|
|`Fscan`|从指定 reader 读取|
|`Fscanf`|根据格式从 reader 读取|
|`Fscanln`|从 reader 读取一行|

 从字符串读取

|函数|说明|
|---|---|
|`Sscan`|从字符串读取|
|`Sscanf`|根据格式从字符串读取|
|`Sscanln`|从字符串读取一行|

输入函数返回成功扫描的参数个数和可能的错误。注意：`Scan` 系列将换行符也视为空格（`Scanln` 除外），`Scanf` 的格式字符串中空白字符会匹配任意空白。


#  6. 自定义类型的格式化（Stringer / Formatter）

### 实现 `Stringer` 接口
类型实现 `String() string` 方法后，使用 ==`%v`、`%s`、`Print`、`Println` 等会调用该方法==。
```go
type User struct {
    Name string
    Age  int
}

func (u User) String() string {
    return fmt.Sprintf("User{Name:%s, Age:%d}", u.Name, u.Age)
}

u := User{"Tom", 18}
fmt.Println(u)  // User{Name:Tom, Age:18}
```

### 实现 `GoStringer` 接口
实现 `GoString() string` 方法后，==%#v会使用该方法输出== Go 语法表示。
```go
func (u User) GoString() string {
    return fmt.Sprintf("User{Name:%q, Age:%d}", u.Name, u.Age)
}
fmt.Printf("%#v\n", u)  // User{Name:"Tom", Age:18}
```

### 实现`Formatter` 接口
类型可以实现 `Format(f State, verb rune)` 方法，==完全控制格式化过程==。
```go
type Point struct{ X, Y int }
func (p Point) Format(f fmt.State, verb rune) {
    switch verb {
    case 'v':
        if f.Flag('+') {
            fmt.Fprintf(f, "(%d,%d)", p.X, p.Y)
        } else {
            fmt.Fprintf(f, "%d %d", p.X, p.Y)
        }
    case 's':
        fmt.Fprintf(f, "[%d,%d]", p.X, p.Y)
    default:
        fmt.Fprintf(f, "%%!%c(Point=%v)", verb, p)
    }
}
```