# 1. 简介
SMTP（Simple Mail Transfer Protocol）是互联网上电子邮件传输的核心协议之一 
==负责将邮件从发送方传递到接收方的邮件服务器==。

C-S模型
基于TCP
请求-响应式

| 端口      | 用途                                  |
| ------- | ----------------------------------- |
| **25**  | SMTP 默认端口，用于 MTA 间通信                |
| **587** | 邮件提交端口，客户端提交邮件时使用，通常要求 STARTTLS 和认证 |
| **465** | 隐式 TLS（SMTPS），用于客户端加密提交             |

# 2. 客户端操作命令
| 命令          | 语法                         | 功能                 |
| ----------- | -------------------------- | ------------------ |
| `HELO`      | `HELO domain`              | 旧式问候，提供域名          |
| `EHLO`      | `EHLO domain`              | 扩展问候，请求扩展列表        |
| `MAIL FROM` | `MAIL FROM:<address> [参数]` | 指定信封发件人            |
| `RCPT TO`   | `RCPT TO:<address> [参数]`   | 指定信封收件人            |
| `DATA`      | `DATA`                     | 开始发送邮件内容           |
| `RSET`      | `RSET`                     | 重置会话，清除信封信息        |
| `VRFY`      | `VRFY <string>`            | 验证用户名/邮箱是否存在（常被禁用） |
| `EXPN`      | `EXPN <list>`              | 展开邮件列表（常被禁用）       |
| `NOOP`      | `NOOP`                     | 无操作，服务器返回 250      |
| `QUIT`      | `QUIT`                     | 结束会话               |

通过 客户端（E代表扩展）`EHLO`    服务器返回支持的ESMTP 扩展命令列表，如下选择：

| 命令                    | 功能             |
| --------------------- | -------------- |
| `STARTTLS`            | 将连接升级为 TLS 加密  |
| `AUTH`                | 客户端认证          |
| `SIZE`                | 声明邮件大小限制       |
| `8BITMIME`            | 支持 8 位 MIME 内容 |
| `PIPELINING`          | 允许管道化发送命令      |
| `BDAT`                | 替代 DATA 的分块传输  |
| `DSN`                 | 投递状态通知         |
| `ENHANCEDSTATUSCODES` | 增强状态码          |
| `SMTPUTF8`            | 支持 UTF-8 地址和头部 |

# 3. 服务端响应状态码
| 代码      | 含义                     |
| ------- | ---------------------- |
| **220** | 服务就绪                   |
| **221** | 服务关闭连接                 |
| **250** | 请求动作完成                 |
| **251** | 用户非本地，将转发              |
| **252** | 无法验证用户，但接受邮件           |
| **235** | 认证成功（AUTH）             |
| **334** | 认证中，要求客户端发送凭据（AUTH 交互） |
| **354** | 开始邮件输入，以 `.` 结束        |
| **421** | 服务不可用，关闭连接             |
| **450** | 邮箱不可用（临时）              |
| **451** | 请求动作中止：本地处理错误          |
| **452** | 系统存储不足                 |
| **500** | 语法错误，命令无法识别            |
| **501** | 参数语法错误                 |
| **502** | 命令未实现                  |
| **503** | 命令顺序错误                 |
| **504** | 命令参数未实现                |
| **530** | 需要认证                   |
| **550** | 邮箱不可用（永久）              |
| **551** | 用户非本地，请尝试转发路径          |
| **552** | 超出存储分配                 |
| **553** | 邮箱名不允许                 |
| **554** | 事务失败                   |
# 4. 邮件Data格式

SMTP 的 DATA部分遵循
1. **Received 字段**：每个经过的 MTA 都会在顶部添加一条 `Received` 头，用于追踪路由。
2. **头部字段**：由字段名、冒号、值组成，一行一个字段，头部与正文之间用空行分隔。
3. **MIME**（多用途 Internet 邮件扩展）：定义附件、HTML 正文、非 ASCII 字符编码等
```text
Received: from mail.example.com (mail.example.com [192.0.2.10])
          by mx.receiver.org (Postfix) with ESMTPS id 4ABCD123
          for <bob@receiver.org>;
          Mon, 7 Sep 2026 10:30:15 +0800 (CST)

From: Alice <alice@example.com>
To: Bob <bob@example.org>
Subject: Meeting
Date: Mon, 7 Sep 2026 10:30:00 +0800
Message-ID: <abc123@example.com>
MIME-Version: 1.0
Content-Type: multipart/mixed;boundary="----=_Part_0_123456"

------=_Part_0_123456
Content-Type: text/plain; charset="utf-8"
Content-Transfer-Encoding: quoted-printable

=E4=BD=A0=E5=A5=BD=EF=BC=8C=E8=BF=99=E6=98=AF=E6=AD=A3=E6=96=87=E3=80=82

------=_Part_0_123456
Content-Type: application/pdf; name="document.pdf"
Content-Transfer-Encoding: base64
Content-Disposition: attachment; filename="document.pdf"

JVBERi0xLjQKJcOkw7zDtsOfCjIgMCBvYmoKPDwvTGVuZ3RoIDMgMCBSL0ZpbHRlci9GbGF0ZURl
Y29kZT4+CnN0cmVhbQp4nF2QPU7DMBSFd57iHVmgJGSKlMRAJxZYGGKJvhLHJvJPCXJbGacSR+Dg
...

```


# 5. 交互流程
## 5.1 建立连接
![[Pasted image 20260907124259.png]]
## 5.2TSL加密与身份验证（可选）

SMTP 本身是不安全的，因为它在传输过程中使用明文传输数据。
为了提高安全性，可以使用以下扩展：
- ==STARTTLS==：将明文连接升级为加密连接，使用 TLS/SSL 加密数据。
- ==扩展命令 AUTH==：通过身份验证机制（如 PLAIN、LOGIN）验证用户身份。

## 5.3 邮件发送
![[Pasted image 20260907124318.png|673]]
## 5.4 关闭连接
![[Pasted image 20260907124355.png]]
# 6. 示例S: 220 mx.example.com ESMTP Postfix
```
C: EHLO client.example.net
S: 250-mx.example.com
   250-PIPELINING
   250-SIZE 52428800
   250-STARTTLS
   250-AUTH PLAIN LOGIN
   250-8BITMIME
   250 SMTPUTF8
   
C: STARTTLS
S: 220 2.0.0 Ready to start TLS
  （TLS 握手）
   
C: AUTH LOGIN
S: 334 VXNlcm5hbWU6
C: dXNlckBleGFtcGxlLmNvbQ==
S: 334 UGFzc3dvcmQ6
C: c2VjcmV0
S: 235 2.7.0 Authentication successful

C: MAIL FROM:<user@example.com> SIZE=1500
S: 250 2.1.0 OK

C: RCPT TO:<recipient@example.org>
S: 250 2.1.5 OK

C: DATA
S: 354 End data with <CR><LF>.<CR><LF>
C: From: User <user@example.com>
   To: Recipient <recipient@example.org>
   Subject: Test
   Date: Mon, 7 Sep 2026 10:30:00 +0800
   
   Hello, this is a test.
   .
S: 250 2.0.0 OK: queued as ABCD1234

C: QUIT
S: 221 2.0.0 Bye
```