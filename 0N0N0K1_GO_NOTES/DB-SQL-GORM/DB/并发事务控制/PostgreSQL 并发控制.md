# 1. MVCC
## 元组头部字段
![[Pasted image 20260924233032.png]]
1. **t_xmin**： 字段记录插入该行的事务ID  
2. **t_xmax**： 字段记录删除或更新该行的事务ID
3. **t_ctid**： 不更新指向自己，反之指向新行ctid；  
4. **t_infomask**： 中的 Hint Bits 缓存 xmin/xmax 对应事务的提交状态，避免频繁查 CLOG 日志
## 快照（Snapshot）
由事务管理器提供（负责分配txid，保存当前运行事务的有关信息）
```sql
SELECT txid_current_snapshot()
    -- 查看
    -- xmin:xmax:xip_list
```
快照包含三个核心要素：
- ==快照xmin==（当前活跃事务中最小事务ID）
- ==快照xmax==（下一个待分配事务ID）
- ==快照xip 数组==（当前所有活跃事务ID列表）
## 可见性判断

>[!tip] 先查 Hint Bits（提示位，元组头部t_infomask中，包含t_xmin与t_xmax对应事务的状态，用于减少对CLOG的访问），若未缓存则查 CLOG 提交日志（记录每个事务的提交状态（IN_PROGRESS / COMMITTED / ABORTED / SUB_COMMITTED））

MVCC 可见性判断规则
先对检查 元组头部xmin 事务：
1. 元组头部xmin 事务已回滚→元组不可见  
2. 元组头部xmin 事务为自己→进入元组头部xmax 判断
3. 元组头部xmin 事务不为自己且正在运行→元组不可见  
4. 元组头部xmin 事务已提交→进入元组头部xmax 判断  
 
检查 元组头部xmax 事务:
1. 元组头部xmax = 0 没有删除/更新/锁→元组可见
2. 元组头部xmax= 当前事务 ID
     如果是当前事务执行的 DELETE 或 UPDATE，且命令已经生效 → 元组不可见
     如果只是当前事务加的锁 → 元组仍可见。
3. 元组头部xmax 在 xip_list 中
     元组头部xmax事务在快照创建时正在运行→ 元组仍可见
4. 元组头部xmax 不在 xip_list 中
     1. 元组头部xmax >= 快照xmax
         说明元组头部xmax事务 在快照创建时尚未开始 → 元组仍可见
    2. 元组头部xmax < 快照xmax 且不在 xip_list，查 CLOG 提交日志
         已提交，删除/更新生效 → 元组不可见
         已回滚，删除/更新无效 → 元组可见
     3. 元组头部xmax <快照xmin，同样查 CLOG 提交日志判断提交还是回滚
 
## 隔离级别

| 级别               | 说明                                                                                        | 解决问题               |
| ---------------- | ----------------------------------------------------------------------------------------- | ------------------ |
| READ COMMITTED   | ==这是 PostgreSQL 的默认隔离级别==；每次查询语句开始时重新获取快照，因此同一事务内多次 SELECT 可能返回不同结果，一个事务只能读取到其他已提交事务修改的数据 | 避免脏读，存在不可重复读和幻读    |
| REPEATABLE READ  | 事务启动时获取一次快照并全程复用，保证多次读取结果一致                                                               | 避免脏读与不可重复读，存在幻读    |
| SERIALIZABLE     | 在 REPEATABLE READ 基础上增加谓词锁SIREAD与读写冲突检测                                                   | 完全避免了脏读、不可重复读和幻读问题 |
| READ UNCOMMITTED | 不支持                                                                                       |                    |
谓词锁SIREAD：有元组、页、表三个级别
- 索引的SIREAD锁为页级，顺序扫描（无论是否有索引、是否有where条件）时一律创建表级
- 检测到冲突时，先提交的事务真正执行，后提交的报错回滚/中止
- 假阳性冲突：当writer一个实际未读的元组但是因为顺序扫描时SIREAD锁级别而全表元组上锁的出现冲突



- 读-写冲突
         在SERIALIZABLE下，一个事务先读取了某元组，在未提交前出现其他事务将该元组更新，导致事务读取的与数据库中实际的不一样，没有保证事务逻辑依次进行
         ![[Pasted image 20260925112732.png]]
丢失更新(写-写冲突)
         多个事务并发更新同一行时出现
         pgsql处理策略流程如下
![[Pasted image 20260925104522.png]]
# 锁机制
## 表级锁（Table-Level Locks）
表级锁作用于整个数据表。当需要对整个表进行操作，如`CREATE INDEX`或`DROP TABLE`时，会使用表级锁

| 锁                           | 说明                                                                  |
| --------------------------- | ------------------------------------------------------------------- |
| ACCESS SHARE LOCK           | 用于数据查询(SELECT)，与ACCESS EXCLSIVE锁冲突                                  |
| ROW SHARE LOCK              | 用于SELECT FOR UPDATE或SELECT FOR SHARE，与EXCLUSIVE和ACCESS EXCLUSIVE锁冲突 |
| ROW EXCLUSIVE LOCK          | 用于数据的更新、插入和删除操作，与SHARE、SHARE ROW EXCLUSIVE和ACCESS EXCLUSIVE锁冲突      |
| SHARE LOCK                  | 用于创建索引（CREATE INDEX），防止并发的数据变更                                      |
| SHARE ROW EXCLUSIVE LOCK    | 用于某些不会自排他的操作，比如创建触发器                                                |
| EXCLUSIVE LOCK              | 用于防止并发的数据变更和读取操作，只允许并发的ACCESS SHARE锁                                |
| ACCESS EXCLUSIVE LOCK       | 用于TRUNCATE、DROP TABLE等DDL操作，与其他所有锁冲突                                |
| SHARE UPDATE EXCLUSIVE LOCK | 用于VACUUM和某些ALTER TABLE操作，防止并发的schema改变和VACUUM命令                     |
## 行级锁（Row-Level Locks）
行级锁是PostgreSQL中最细粒度的锁。它们作用于数据表中的单个行，允许多个事务同时修改不同的行，从而最大化并发性

| 锁                 | 说明                                        |
| ----------------- | ----------------------------------------- |
| FOR UPDATE        | 对整行进行更新，包括删除行，阻止其他事务对行的读取和更新              |
| FOR NO KEY UPDATE | 对除主(唯一)键外的字段更新，对行加锁，但允许其他事务在不锁定键值的情况下进行更新 |
| FOR SHARE         | 读该行，不允许对行进行更新，阻止其他事务对行的更新                 |
| FOR KEY SHARE     | 读该行的键值，但允许对除键外的其他字段更新，主要用于外键检查            |
## 页级锁（Page-Level Locks）
页级锁介于行级锁和表级锁之间，作用于数据表中的单个数据页。它们常用于索引操作和批量数据修改,页级锁用于控制对数据页的并发访问

| 锁             | 说明                  |
| ------------- | ------------------- |
| WALInsertLock | 向WAL缓冲区写入WAL记录时需要的锁 |
| WALWriteLock  | 确保WAL数据被刷入磁盘的锁      |
| ProcArrayLock | 用于追踪正在运行的后端进程和事务    |
## 咨询锁（Advisory Locks）

询锁是一种用户控制的锁，用于应用程序级别的锁定。它们不与特定的数据库对象关联，而是由应用程序根据需要获取和释放
咨询锁可以是会话级别的或事务级别的，允许用户在不同的进程或事务之间进行协调
![[Pasted image 20260923224044.png]]
## 死锁（Deadlocks）

死锁发生在两个或多个事务相互等待对方持有的锁时。
PostgreSQL有参数如`lock_timeout`、`deadlock_timeout`和`log_lock_waits`来控制死锁的检测和处理。  
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/a4b6365112a34f6192e57f7a00fd5695.png)