# 功能描述
## 变量介绍
添加两个新global级别的变量 kundb_kill_idle_transaction 与 kundb_special_users
### kundb_kill_idle_transaction
变量范围: global
数据类型: int
默认单位: 秒(second)
默认值: 0
含义: 此变量的值大于0时，此变量对由**kundb_special_users**指定的用户生效。当这些用户开启了事务(begin, start transaction)，并在事务中第一次执行以下4种语句后：
- delete
- update
- select for update
- select for share/ select lock in share mode
  
当前session的一个特定flag将被置为true，此flag为true时，要求 上一条sql执行完毕并返回给客户端（无论是成功执行还是失败）到 gate收到下一条sql 之间的时间少于**kundb_kill_idle_transaction**秒。
否则，该事务将被强制回滚，并且客户端与gate的连接也将被强制断开。此flag将在commit/rollback时被reset为false。

### kundb_special_users
变量范围: global
数据类型: string
默认值: ""
含义：当**kundb_kill_idle_transaction**生效时，只有**kundb_special_users**里含有的User会受影响，换句话说，不在**kundb_special_users**的用户建立的所有连接均不会被gate kill掉。

注意
- 此变量可以接受指定多个User，每两个User之间必须用单个空格分开。例如
set kundb_special_users = 'bob jim alexandra'意味着bob,jim,alexandra这三个人会受**kundb_kill_idle_transaction**的影响，换句话说他们是受害者。
- 此变量仅对被set之后新建的conn有效，例如你想将john加入到受害者联盟的话，在用set语句设置了用户之后，不会对当前john已经连接到gate上的连接有任何作用。

## 效果显示
![image](https://github.com/user-attachments/assets/1150720f-3432-4df1-a877-9ff318bc87a4)

    
