# 1. 作用
用于处理错误的基础工具包。它提供了==创建、包装、检查错误的功能==，是现代 Go 错误处理的核心
> **`errors.New` 造错误，`fmt.Errorf` + `%w` 包错误，`errors.Is` 找错误，`errors.As` 抓错误，`Unwrap` 剥错误。**
# 2. 核心函数
#### 2.1 错误与创建
```go
//  error 的接口
type error interface {
    Error() string //实现返回错误信息的方法
}

//每次调用 errors.New 都会创建一个新的错误实例，它们指针不同。
func New(text string) error

err := errors.New("something went wrong")

//fmt.Errorf —— 格式化 + 包装错误
func Errorf(format string, a ...any) (err error)
//  %v	仅嵌入错误信息，不建立链
//  %w	包装错误，建立错误链（可被 Is/As 追溯）

// 不包装，只是拼接字符串
err1 := fmt.Errorf("read failed: %v", io.EOF)

// 包装，建立错误链
err2 := fmt.Errorf("read failed: %w", io.EOF)
// 再次包装
err3 := fmt.Errorf("service call failed: %w", err2)
```
 ==关键区别：只有 %w 包装的错误才能被 errors.Is 和 errors.As 识别。==
一个可被包装的错误需要实现：
```go
//如果 err 的类型包含一个返回错误的 Unwrap 方法，则 Unwrap 返回对 err 调用 Unwrap 方法的结果。否则，Unwrap 返回 nil
func Unwrap(err error) error

type unwrapper interface {
    Unwrap(err error) error
}
```

#### 2.2 错误链操作
 
 1. errors.Is 
```go
//检查错误树中是否存在与目标匹配的错误。目标必须具有可比性
func Is(err, target error) bool
```
**原理**：沿着错误链逐层 `Unwrap`，直到找到匹配或链结束。
```go
// 示例
base := io.EOF
wrapped := fmt.Errorf("read error: %w", base)
double := fmt.Errorf("service error: %w", wrapped)

fmt.Println(errors.Is(double, io.EOF))  // true
```
**自定义匹配**：让自定义错误实现 `Is(target error) bool` 方法。
```go
type NotFoundError struct { Resource string }

func (e *NotFoundError) Error() string {
    return fmt.Sprintf("%s not found", e.Resource)
}
// 自定义 Is 行为：只要 target 也是 NotFoundError，就算匹配
func (e *NotFoundError) Is(target error) bool {
    _, ok := target.(*NotFoundError)
    return ok
}
```

2. errors.As
```go
//函数会在错误树中查找第一个与目标匹配的错误，如果找到，则将目标设置为该错误值并返回 true；否则，返回 false。
func As(err error, target any) bool

```
**自定义 As**：实现 `As(target any) bool` 方法。
```go
func (e *MyError) As(target any) bool {
    if t, ok := target.(**MyError); ok {
        *t = e
        return true
    }
    return false
}
```
# 3. 最佳实践与常见坑
#### 3.1 自定义错误类型
```go
type AppError struct {
    Code    int
    Message string
    Cause   error  // 底层原因
}

func (e *AppError) Error() string {
    if e.Cause != nil {
        return fmt.Sprintf("[%d] %s: %v", e.Code, e.Message, e.Cause)
    }
    return fmt.Sprintf("[%d] %s", e.Code, e.Message)
}

// 实现 Unwrap，让 errors.Is/As 能追溯 Cause
func (e *AppError) Unwrap() error {
    return e.Cause
}

// 使用
err := &AppError{
    Code:    404,
    Message: "user not found",
    Cause:   sql.ErrNoRows,
}

if errors.Is(err, sql.ErrNoRows) {
    // 能匹配到！
}
```

####  3.2 哨兵错误（Sentinel Errors）
```go
//预定义一些全局错误变量，作为判断标识
var (
    ErrNotFound    = errors.New("not found")
    ErrInvalidInput = errors.New("invalid input")
    ErrUnauthorized = errors.New("unauthorized")
)

// 使用
if err == ErrNotFound { ... }           // Go 1.13 前的方式
if errors.Is(err, ErrNotFound) { ... }  // Go 1.13+ 推荐方式
```

> 哨兵错误变量命名惯例：`ErrXxx`

#### 3.3 使用建议
##### ✅ Do

| 实践                                   | 说明                             |
| :----------------------------------- | :----------------------------- |
| 用 `fmt.Errorf("...: %w", err)` 添加上下文 | 保留原始错误信息，同时建立链                 |
| 用 `errors.Is` 代替 `err == xxx`        | 支持错误链                          |
| 用 `errors.As` 提取具体类型                 | 比类型断言 `.(type)` 更健壮            |
| 定义哨兵错误                               | `var ErrXxx = errors.New(...)` |
| 自定义错误类型时实现 `Unwrap()`                | 让调用方能追溯 Cause                  |

##### ❌ Don't

| 反模式                                        | 原因                                                       |
| :----------------------------------------- | :------------------------------------------------------- |
| `fmt.Errorf("...%v", err)` 后还想 `errors.Is` | `%v` 不建立链，`Is` 会失败                                       |
| 到处 `log.Fatal` / `panic`                   | Go 哲学：错误向上传，顶层处理                                         |
| 错误信息首字母大写、带标点                              | 惯例：`errors.New("file not found")`，不是 `"File not found."` |
| 忽略错误 `_ = doSomething()`                   | 除非你真的确定可以忽略                                              |







