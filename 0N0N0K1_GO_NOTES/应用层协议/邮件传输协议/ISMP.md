# 1. 简介
IMAP（Internet Message Access Protocol）是一种用于从邮件服务器访问和管理电子邮件的协议
==允许用户在服务器上管理邮件，支持多设备同步和高级邮件操作==

C-S模型
基于TCP
请求-响应式

| 端口      | 说明              |
| ------- | --------------- |
| **143** | 默认端口，数据不加密      |
| **993** | 使用SSL/TLS加密传输端口 |


# 2.客户端操作命令

| 命令             | 用途                   |
| -------------- | -------------------- |
| `CAPABILITY`   | 查询服务器支持的功能           |
| `NOOP`         | 空操作，保持连接活跃           |
| `LOGOUT`       | 退出登录，关闭连接            |
| `UID`          | 作为前缀，使用**UID**而非序号操作 |
| `STARTTLS`     | 在明文连接上启用TLS加密        |
| `AUTHENTICATE` | 使用SASL机制安全认证         |
| `LOGIN`        | 明文用户名密码登录（需加密通道）     |
| `SELECT`       | 选择邮箱（读写）             |
| `EXAMINE`      | 选择邮箱（只读）             |
| `CREATE`       | 创建新邮箱                |
| `DELETE`       | 删除邮箱                 |
| `RENAME`       | 重命名邮箱                |
| `SUBSCRIBE`    | 订阅邮箱                 |
| `UNSUBSCRIBE`  | 取消订阅                 |
| `LIST`         | 列出邮箱                 |
| `LSUB`         | 列出已订阅邮箱              |
| `STATUS`       | 查询邮箱状态（邮件数/未读数等）     |
| `APPEND`       | 追加邮件到指定邮箱            |
| `FETCH`        | 获取邮件数据（头部/正文/标志等）    |
| `STORE`        | 修改邮件标志（已读/删除等）       |
| `COPY`         | 复制邮件到另一邮箱            |
| `SEARCH`       | 按条件搜索邮件              |
| `EXPUNGE`      | 永久删除标记为`\Deleted`的邮件 |
| `CHECK`        | 请求服务器执行检查点           |
| `CLOSE`        | 关闭邮箱并永久删除已标记邮件       |

# 3.服务端响应

IMAP的每条响应都由三部分组成：
1. **状态类型 (OK/NO/BAD)**、
2. **机器可读响应码**（位于方括号`[]`中），
3. **人类可读的描述文本**

|状态类型|是否可带标签|核心含义|
|---|---|---|
|**OK**|是|**成功**。命令成功完成[](https://reference.aspose.com/email/zh/net/aspose.email.clients.imap/imapstatuscode/)。|
|**NO**|是|**失败**。命令执行失败[](https://reference.aspose.com/email/zh/net/aspose.email.clients.imap/imapstatuscode/)。|
|**BAD**|是|**错误**。命令语法错误或协议错误[](https://reference.aspose.com/email/zh/net/aspose.email.clients.imap/imapstatuscode/)。|
|**PREAUTH**|否 (未带标签)|**预认证**。连接已通过外部方式认证，无需再执行`LOGIN`命令[](https://reference.aspose.com/email/zh/net/aspose.email.clients.imap/imapstatuscode/)。|
# 4. 交互流程

## 4.1 建立连接
![[Pasted image 20260907155209.png]]
## 4.2 TSL加密与身份认证（可选）


```
S: * OK [CAPABILITY IMAP4rev1 STARTTLS AUTH=PLAIN LOGINDISABLED] Dovecot ready.
   (通告支持 STARTTLS，同时 LOGINDISABLED 表示登录前必须先加密)
   
C: A001 CAPABILITY
S: * CAPABILITY IMAP4rev1 STARTTLS AUTH=PLAIN LOGINDISABLED
S: A001 OK CAPABILITY completed    //(服务器确认 STARTTLS 可用)
  
C: A002 STARTTLS
S: A002 OK Begin TLS negotiation now
   
 ========== TLS 加密通道建立 ==========
 
//TLS 建立后，必须重新发送 CAPABILITY 命令
//因为服务器能力列表可能已变化，且 STARTTLS 自身不应再出现
C: A003 CAPABILITY
S: * CAPABILITY IMAP4rev1 AUTH=PLAIN AUTH=LOGIN              // AUTH=PLAIN 现在可用
S: A003 OK CAPABILITY completed
   
```

## 4.3 邮件访问和管理
![[Pasted image 20260907155857.png]]
## 4.4 关闭连接
![[Pasted image 20260907155945.png]]

# 5. 示例

```
S: * OK [CAPABILITY IMAP4rev1 STARTTLS AUTH=PLAIN] Dovecot ready.
   
C: A001 CAPABILITY
S: * CAPABILITY IMAP4rev1 LITERAL+ SASL-IR LOGIN-REFERRALS ID ENABLE IDLE AUTH=PLAIN
S: A001 OK Capability completed.
   
C: A002 LOGIN "alice" "mysecretpassword"
S: A002 OK [CAPABILITY IMAP4rev1 ...] User alice authenticated.

C: A003 SELECT INBOX
S: * 172 EXISTS                                       // (收件箱共有 172 封邮件)
S: * 5 RECENT                                        //(其中有 5 封新邮件)
S: * OK [UNSEEN 12] Message 12 is first unseen             //(第一条未读邮件的序号是 12)
S: * OK [UIDVALIDITY 3857529045] UIDs valid    //(UID 有效性值，若变化则之前缓存的 UID 均失效)
S: * OK [UIDNEXT 442] Predicted next UID             // (下一个可分配的 UID 是 442)
S: * FLAGS (\Answered \Flagged \Deleted \Seen \Draft)
S: * OK [PERMANENTFLAGS (\Answered \Flagged \Deleted \Seen \Draft \*)]
S: A003 OK [READ-WRITE] SELECT completed.
   
C: A004 FETCH 1:* (FLAGS)                // (获取所有邮件的标志，用于快速了解哪些已读/删除等)
S: * 1 FETCH (FLAGS (\Seen))
S: * 2 FETCH (FLAGS (\Seen \Answered))
S: * 3 FETCH (FLAGS (\Recent))
S: * 4 FETCH (FLAGS (\Seen))
... (省略中间部分)
S: * 172 FETCH (FLAGS (\Deleted))
S: A004 OK FETCH completed.                           //(返回每个邮件的序号和标志)
 
C: A005 FETCH 12 BODY.PEEK[TEXT]          //(获取第 12 封邮件的正文文本内容，但不要标记为已读)
S: * 12 FETCH (BODY[TEXT] {512}              //(服务器开始发送字面量数据，大小为 512 字节)
S: Dear Alice, ...
   (此处是完整的邮件正文，共 512 字节)
S: )
S: A005 OK FETCH completed.                      //(正文传输完成)
  
C: A006 STORE 12 +FLAGS (\Seen)                     //(将第 12 封邮件标记为已读)
S: * 12 FETCH (FLAGS (\Seen \Recent))                    //(服务器主动返回更新后的标志)
S: A006 OK STORE completed.                            //(标志修改成功)
   
C: A007 SEARCH UNSEEN                               //(搜索所有未读邮件)
S: * SEARCH 3 15 22 31 44 57 63 89 101 120 135 148 165
S: A007 OK SEARCH completed.

C: A008 STATUS INBOX (MESSAGES UNSEEN)               // (再次查询收件箱状态，确认未读数)
S: * STATUS INBOX (MESSAGES 172 UNSEEN 13)              //(当前总邮件数 172，未读 13)
S: A008 OK STATUS completed.

C: A009 LOGOUT                                         // (准备退出)
S: * BYE IMAP4rev1 Server logging out
S: A009 OK LOGOUT completed.                        //(会话结束，连接关闭)
   
```