
[sql标准库详解]( https://mp.weixin.qq.com/s/ojDRfrotU8ByOTIYFZxF0g)

[mysql驱动详解](https://mp.weixin.qq.com/s?__biz=MzkxMjQzMjA0OQ==&mid=2247484744&idx=1&sn=d315ce9c80a502a35677595638d450bb&chksm=c0860182c6b4db27485558207937efbbbdd29551f13e6d679950409b1df22f8c81213bfc5776&scene=126&sessionid=1789867985&subscene=7&clicktime=1789867997&enterid=1789867997#rd)

![[Pasted image 20260921103622.png]]

```
 业务代码
    │
    │   import "database/sql"
    │   匿名导入驱动包，使用其init()
    ▼ 
database/sql（标准库）
    │  
    │  连接池、事务、上下文取消、结果集遍历
    │  通过 Driver 接口调用驱动
    │
    ▼
  驱动包
    │
    │  MySQL 协议解析、网络通信、认证
    │ 
    ▼
数据库服务器
```

database/sql：
- 连接池管理（`SetMaxOpenConns` 等）
- 事务的 `Begin`/`Commit`/`Rollback` 流程控制
- `context.Context` 超时/取消

驱动：
- 占位符 `?` → MySQL 协议参数的转换
- MySQL 握手、认证、TLS、协议编解码
- 数据类型 ↔ Go 类型的转换