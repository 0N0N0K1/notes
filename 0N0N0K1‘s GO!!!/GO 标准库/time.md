# 1. 作用
处理时间和日期的核心包，提供了==时间的表示、计算、格式化、解析、定时器==等功能

# 2. 关键类型
#### 2.1. time.Time
time.Time 是一个结构体，值类型，精确到纳秒，同时包含了时区信息和单调时钟读数内部存储的是绝对时间（从基准点开始的纳秒数）以及一个 *Location 指针。
在不同时区下显示的时间字符串不同，但绝对时间相同。使用 In(loc) 方法可以将时间转换为指定时区的表示，而不改变绝对时间。
使用时间的程序通常应该将时间存储和传递为值，而不是指针。也就是说，时间变量和结构体字段应该是 time.Time类型，而不是 *time.Time 类型。
```go
type Time struct {  
    ext  int64  
    wall uint64  
    ext  int64    
    loc *Location  
}
```
 创建
```go
now := time.Now()          // 当前本地时间
t := time.Date(2025, 3, 10, 12, 30, 0, 0, time.UTC) // 指定时间
t2 := time.Unix(1741583400, 0) // 从 Unix 时间戳创建
```
比较操作
```go
t1 := time.Now()
t2 := t1.Add(time.Hour)
fmt.Println(t1.Before(t2)) // true
fmt.Println(t1.Equal(t2))  // false
```

获取年、月、日、时、分、秒等
```go
year, month, day := t1.Date()
hour, min, sec := t1.Clock()
```
零值
```go
//time.Time 的零值是 January 1, year 1, 00:00:00 UTC，通常用 t.IsZero() 检查。
func (t Time) IsZero() bool
```

#### 2.2 time.Duration
time.Duration 是 int64 的类型别名，表示纳秒数。最长可表示约 290 年。
常用单位常量：
```go
type Duration int64

const (
    Nanosecond  Duration = 1
    Microsecond          = 1000 * Nanosecond
    Millisecond          = 1000 * Microsecond
    Second               = 1000 * Millisecond
    Minute               = 60 * Second
    Hour                 = 60 * Minute
)
```
支持算术运算，如 d := 2*time.Hour + 30*time.Minute,运算数值为 int64 类型
支持 String() 方法，会格式化为易读的字符串（如 "2h30m0s"）。

#### 2.3. time.Location
time.Location 表示时区（可能包含夏令时规则）。常用获取方式：

```Go
time.UTC //UTC 时区。

time.Local//本地时区（由系统环境变量 TZ 决定）。

time.LoadLocation(name)//从 IANA 时区数据库加载，如 "Asia/Shanghai"。

time.FixedZone(name, offset)//创建固定偏移的时区。

```

# 3. 格式化——将 time.Time解析为字符串
Go 使用特殊的参考时间 Mon Jan 2 15:04:05 MST 2006 
用于 Time.Format 和 time.Parse 的预定义布局
这与常见的 yyyy-MM-dd HH:mm:ss 不同
```go
//格式化函数
func (t Time) Format(layout string) string

//常用布局常量
const (
    Layout      = "01/02 03:04:05PM '06 -0700" // 美式
    ANSIC       = "Mon Jan _2 15:04:05 2006"
    UnixDate    = "Mon Jan _2 15:04:05 MST 2006"
    RubyDate    = "Mon Jan 02 15:04:05 -0700 2006"
    RFC822      = "02 Jan 06 15:04 MST"
    RFC822Z     = "02 Jan 06 15:04 -0700"
    RFC850      = "Monday, 02-Jan-06 15:04:05 MST"
    RFC1123     = "Mon, 02 Jan 2006 15:04:05 MST"
    RFC1123Z    = "Mon, 02 Jan 2006 15:04:05 -0700"
    RFC3339     = "2006-01-02T15:04:05Z07:00"
    RFC3339Nano = "2006-01-02T15:04:05.999999999Z07:00"
    Kitchen     = "3:04PM"
    Stamp      = "Jan _2 15:04:05"
    StampMilli = "Jan _2 15:04:05.000"
    StampMicro = "Jan _2 15:04:05.000000"
    StampNano  = "Jan _2 15:04:05.000000000"
)
//自定义格式示例：
t := time.Now()
fmt.Println(t.Format("2006-01-02 15:04:05"))
fmt.Println(t.Format("2006/01/02 03:04:05 PM"))
```
# 4. 解析——将字符串解析为 time.Time
使用 UTC 时区： time.Parse(layout, value string) 。
使用本地时区：time.ParseInLocation(layout, value, time.Local)。
```go
//解析函数
func Parse(layout, value string) (Time, error)

//示例
t, err := time.Parse("2006-01-02 15:04:05", "2025-03-10 12:30:00")
if err != nil {
    log.Fatal(err)
}
// 指定时区
loc, _ := time.LoadLocation("Asia/Shanghai")
t, err = time.ParseInLocation("2006-01-02 15:04:05", "2025-03-10 12:30:00", loc)
```
# 5. 时间计算
```go
Add(d Duration) Time //加上一个时间段。
AddDate(years, months, days int) Time //按日历加减年月日
Sub(u Time) Duration //两个时间点之间的差值。
Since(t Time) Duration //time.Now().Sub(t) 的快捷方式。
Until(t Time) Duration //t.Sub(time.Now()) 的快捷方式。
func (t Time) Round(d Duration) Time//Round 函数返回将 t 四舍五入到最接近的 d 的倍数（自零点起）的结果。对于中间值，向上取整。如果 d <= 0，Round 函数返回的 t 值会去除单调时钟
```
# 6. time.Timer 和 time.Ticker
1.  time.Timer（一次性定时器）
```go
//当 Timer 超时时，除非 Timer 是由 AfterFunc 创建的，否则当前时间将发送到 C
type Timer struct {  
    C         <-chan Time  
    initTimer bool  
}

timer := time.NewTimer(2 * time.Second)

<-timer.C          // 阻塞直到定时器触发

//等效与使用time.After
//func After(d Duration) <-chan Time {  
//    return NewTimer(d).C  
//}
select {
case <-time.After(time.Second):
    fmt.Println("Timeout")
}

//AfterFunc 会等待指定的时间过去，然后在它自己的 goroutine 中调用 f。它返回一个 Timer 对象 ，可以使用其 Stop 方法取消调用。返回的 Timer 对象的 C 字段不会被使用，其值为 nil
func AfterFunc(d Duration, f func()) *Timer


timer.Stop() //可以停止定时器，若已触发则返回 false。

timer.Reset(d) /可以重置定时器。

```
2.  time.Ticker 与 time.Tick（周期性定时器）
```go
type Ticker struct {  
    C          <-chan Time // The channel on which the ticks are delivered.  
    initTicker bool  
}
ticker := time.NewTicker(500 * time.Millisecond)
defer ticker.Stop()
for range ticker.C {
    fmt.Println("Tick")
}
//拥有stop和reset两种方法操做Ticker
//Ticker 的通道不会堆积事件如果消费者处理时间超过间隔,tick 会被丢弃

```

```go
//相对Ticker,tick不能reset和stop
func Tick(d Duration) <-chan Time
```
# 7. 其他常用函数
```go 
time.Sleep(d Duration) //暂停当前 goroutine 指定时间。

time.After(d Duration) <-chan Time  //返回一个在 d 时间后发送当前时间的通道，常用于超时控制。

time.Tick(d Duration) <-chan Time //返回一个周期性发送时间的通道
```

# 8. 易错点

操作系统同时提供“墙钟”（会因时钟同步而发生变化）和“单调时钟”（不会发生变化）
墙钟用于报时，单调时钟用于计时。
time.Time 对象同时包含墙钟读数和单调时钟读数
后续的报时操作使用墙钟读数，而后续的计时操作（比较和减法等）使用单调时钟读数


1. 不要比较 time.Time 的指针或使用 == 比较包含单调时钟的时间
    如果希望比较两个时间点的墙上时钟是否相等，应先剥离单调时钟：t.Round(0) 可以移除单调时钟。
    == 除了时间本身，它还会比较两个“附属品”：Location（时区位置）Monotonic Clock（单调时钟读数）使用 t1.Equal(t2) 来代替 t1 == t2。它会忽略时区归属和单调时钟读数，只比较“这个时刻是否代表同一时间点”
    
2. 不能作为 Map 或 DB 的 Key , 如果你把一个含有 Local 时区的 time.Time 存进 Map 作为 Key，换了一台服务器或者在不同环境下重启程序，时区字段可能发生细微变化
   
3. 解析时间字符串时注意时区

4. 避免使用 time.Tick，容易导致内存泄漏==是错的==
   从 Go 1.23 开始，垃圾回收器可以回收未被引用的 Ticker，即使它们没有被停止。因此，不再需要 Stop 方法来帮助垃圾回收器。既然 Tick 可以满足需求，就没有理由再选择 NewTicker 了。

5. 格式化布局容易出错 参考时间是 Mon Jan 2 15:04:05 MST 2006

6. AddDate 的月末处理

7. 时区数据库：在容器或交叉编译环境中，时区数据可能不可用


