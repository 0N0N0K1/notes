# 1. 简介
POP3（Post Office Protocol version 3）是一种用于从邮件服务器下载电子邮件的协议。
==允许用户通过客户端（如 Outlook）从服务器获取邮件，并将邮件存储到本地设备。==

C-S模型
基于TCP
请求-响应式

| 端口      | 用途                                   |
| ------- | ------------------------------------ |
| **110** | POP 默认端口。可以通过 `STLS` 命令升级为显式 TLS加密端口 |
| **995** | 隐式 TLS加密端口                           |


# 2. 客户端操作命令

客户端状态：
- **AUTHORIZATION 状态**：客户端需要证明身份，通常用 `USER` 和 `PASS` 命令，或 `APOP`、`AUTH`。
- **TRANSACTION 状态**：身份验证通过后，客户端可以列出、读取、标记删除邮件。
- **UPDATE 状态**：==客户端发送 `QUIT` 后进入，服务器执行实际的删除操作，然后关闭连接==。

| 命令                     | 状态            | 说明                                                                                                                            |
| ---------------------- | ------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `USER <name>`          | AUTHORIZATION | 提供用户名                                                                                                                         |
| `PASS <password>`      | AUTHORIZATION | 提供密码，明文传输                                                                                                                     |
| `APOP <name> <digest>` | AUTHORIZATION | 使用 MD5 摘要认证，避免明文密码                                                                                                            |
| `AUTH <mechanism>`     | AUTHORIZATION | SASL 认证扩展（RFC 1734/5034）                                                                                                      |
| `STAT`                 | TRANSACTION   | 返回邮箱中邮件总数和总大小                                       C: STAT<br>S: +OK 2 320                                                   |
| `LIST [msg]`           | TRANSACTION   | 列出邮件编号和大小，或指定邮件大小                         C: LIST<br>S: +OK 2 messages (320 octets)<br>S: 1 120<br>S: 2 200<br>S: .           |
| `RETR <msg>`           | TRANSACTION   | 下载指定邮件全文                                                        C: RETR 1<br>S: +OK 120 octets<br>S: <邮件原始内容，按行传输><br>S: .    |
| `DELE <msg>`           | TRANSACTION   | 标记指定邮件为删除                                                     C: DELE 1<br>S: +OK message 1 deleted                           |
| `NOOP`                 | TRANSACTION   | 空操作，服务器应返回 `+OK`                                                                                                              |
| `RSET`                 | TRANSACTION   | 撤销所有 `DELE` 标记                                                                                                                |
| `TOP <msg> <n>`        | TRANSACTION   | 返回邮件头和正文前 n 行，常用于预览                        C: TOP 1 10<br>S: +OK<br>S: <邮件头部><br>S: <空行><br>S: <正文前 10 行><br>S: .             |
| `UIDL [msg]`           | TRANSACTION   | 返回邮件的唯一标识符，用于客户端去重                      C: UIDL<br>S: +OK<br>S: 1 whqtswO00WBw418f9t5JxYwZ<br>S: 2 QhdPYR:00WBw1Ph7x7<br>S: . |
| `QUIT`                 | 任意            | 退出并更新状态                                                                                                                       |

服务器可以在欢迎消息中声明支持的扩展：
S: +OK POP3 server ready <1896.697170952@mail.example.com>

|扩展|说明|
|---|---|
|`STLS`|启动 TLS 加密|
|`SASL`|支持 SASL 认证机制|
|`UIDL`|支持唯一标识符|
|`TOP`|支持预览|
|`UTF8`|支持国际化邮箱名和邮件头|
|`EXPIRE`|服务器自动删除邮件的策略|
|`IMPLEMENTATION`|服务器实现信息|

# 3. 服务端响应

1. **单行响应**：以 `+OK` 或 `-ERR` 开头，后面跟可选说明。

2. **多行响应**：第一行是 `+OK`，后续是数据行，最后以单独一行的 `.` 结束。


# 4. 交互流程
## 4.1 建立连接
![[Pasted image 20260907141237.png]]
## 4.2 TSL加密与身份认证（可选）

客户端连接到 110 端口后，发送 `STLS` 命令，服务器确认后双方开始 TLS 握手：

```
C: STLS
S: +OK Begin TLS negotiation
<TLS 握手开始>
```

## 4.3 邮件下载
![[Pasted image 20260907141342.png|661]]
## 4.4 关闭连接
![[Pasted image 20260907141455.png]]

# 5. 示例
```
S: +OK POP3 server ready <1896.697170952@mail.example.com>

C: USER alice
S: +OK User accepted

C: PASS secret
S: +OK alice's maildrop has 2 messages (320 octets)

C: STAT
S: +OK 2 320

C: LIST
S: +OK 2 messages (320 octets)
S: 1 120
S: 2 200
S: .

C: UIDL
S: +OK
S: 1 whqtswO00WBw418f9t5JxYwZ
S: 2 QhdPYR:00WBw1Ph7x7
S: .

C: TOP 1 5
S: +OK
S: From: sender@example.com
S: To: alice@example.com
S: Subject: Hello
S: Date: Mon, 1 Jan 2024 10:00:00 +0000
S:
S: This is the first
S: 5 lines of the body.
S: .

C: RETR 1
S: +OK 120 octets
S: From: sender@example.com
S: To: alice@example.com
S: Subject: Hello
S: Date: Mon, 1 Jan 2024 10:00:00 +0000
S:
S: This is the body of the first email.
S: It may contain multiple lines.
S: .

C: DELE 1
S: +OK message 1 deleted

C: QUIT
S: +OK POP3 server signing off (1 message left)
```
==连接关闭后，第 1 封邮件被真正删除，服务器上只保留第 2 封。==