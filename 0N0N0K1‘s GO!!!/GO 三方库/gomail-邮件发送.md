# 1. 作用
通过 [[SMTP]] 向服务器发送/提交邮件
Package gomail 提供了一个简单的接口，方便高效地撰写邮件和发送邮件
文档地址： https://pkg.go.dev/gopkg.in/gomail.v2#section-documentation

# 2. 导入
```text
go get gopkg.in/gomail.v2
```

# 3. 使用示例

 产生6位数字验证码并发送
```go

// SendAuthMail 向传入参数to发送验证码
func SendAuthMail(to string) error {  
    // 定义配置
    qqEmail := "12345678@qq.com"  
    authCode := "abcdefghijk"  
    smtpHost := "smtp.qq.com"  
    smtpPort := 587  
  
    // 产生验证码
    rnd := rand.New(rand.NewSource(time.Now().UnixNano()))  
    code := fmt.Sprintf("%06d", rnd.Intn(1000000))  
  
    // 构造邮件
    m := gomail.NewMessage()  
    //可以通过参数SetCharset与SetEncoding
    //gomail.SetCharset("ISO-8859-1")
    //gomail.SetEncoding(gomail.Base64)
    
    // 构造邮件From, To, Subject, X-Date
    m.SetHeader("From", qqEmail)  
    m.SetHeader("To", to)  //可指定多个to多发
    m.SetHeader("X-Date": {m.FormatDate(time.Now())})// 标准格式化时间
    m.SetHeader("Subject", "验证码")  
    // 消息主体Body(body支持文本，HTML)
    m.SetBody("text/plain", fmt.Sprintf("您的本次操作的验证码是：\n\n%s\n\n请在2      分钟内使用", code))  
    
    // 添加附件
    m.Attach("这是一张图片.jpg",gomail.Rename("picture.jpg"))
   
    // 构造一个 dialer
    d := gomail.NewDialer(smtpHost, smtpPort, qqEmail, authCode) 
    // 测试连通性并发送消息 
    if err := d.DialAndSend(m); err != nil {  
       return err  
    }  
    
    // 发送成功后进行用Redis缓存2min
    DB.RDB.Set(context.TODO(), "auth:code:"+code, to, time.Minute*2)  
    return nil  
}
```

在HTML格式 Body 中嵌入图片
```go
    // 指定嵌入图片地址
    m.Embed("/path/to/your/image.png") 
    // 注意：默认使用文件名作为 CID
    // 通过 <img> 标签的 src 属性引用该 CID 如果图片文件名为 "image.png"，则 CID 默认为 "image.png"
    m.SetBody("text/html", `<img src="cid:image.png" alt="click me" />`)
```