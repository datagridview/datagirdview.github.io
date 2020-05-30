---
layout: page
title: 高性能MySQL
---

## 架构
他的架构设计把**查询处理以及系统任务**和**数据的存储和提取**相分离。

![架构图](https://i.loli.net/2020/05/25/r4DdlefLk1gFtqY.png)

### 并发控制
通过创建锁，来控制并发的资源占用问题。
#### 读写锁
读锁，也叫共享锁，互相不阻塞。
写锁，也叫排他锁，会阻塞其他的读锁和写锁。
#### 锁粒度
通过区分加锁的数据量，来提升系统的并发量。但是锁操作都会增加系统的开销。所以要在安全性和锁开销之间寻求一个平衡。
##### 两种重要的锁策略
表锁：开销最小的策略。一般采用mysql自己实现的表锁而忽略存储引擎的锁机制。
行锁：最大程度的并发处理，伴随最大的开销。且只在存储引擎中实现。

### 事务
四大特性：
* 原子性：要么全部成功，要么全部失败
* 一致性：从一个一致性转换成另一种一致性
* 隔离性：所做的改变在最终提交之前，对于其他事务来说是不可见的
* 持久性：事务提交之后改动不会丢失

#### 四种隔离级别
较低级别的隔离通常可以执行更高的并发，开销也更低。
* read uncommitted（未提交读）：很少使用，问题很多
* read uncommitted（提交读/不可重复读）：大多数数据库系统的默认级别（Mysql不是），其他事务执行两次相同的读可能得到不一样的结果。
* repeatable read（可重复读）：MySQL默认隔离级别。解决脏读问题，会有幻行的问题（多了一行）。Mysql已经解决了这一问题。
* serializable（串行）：强制事务串行执行。对读取的所有行都加锁。

避免不可重复读锁行就行
避免幻读锁表就行

#### 多版本并发控制MVCC
行级锁的变种，很多情况下避免了加锁操作，开销更低。

InnoDB的MVCC通过在每一行后面保存两个隐藏的列来实现，一个是行的创建时间，一个是行的过期时间（版本号）

### 引擎
#### InnoDB
数据存储在表空间中。表是基于聚簇索引建立的。
#### MyISAM
不支持事务和行级锁，崩溃后无法恢复。

MYD和MYI文件存储，分别存储数据文件和索引文件。


## 服务器性能剖析
性能优化并不是降低cpu利用率、也不是提升每秒查询量
### 应用程序的性能剖析
对任何需要消耗时间的任务都可以进行性能剖析。性能瓶颈可能有很多的影响因素：
* 外部资源。调用了外部的服务或者搜索引擎
* 应用处理大量的数据，分析一个超大规模的XML文件
* 在循环中执行昂贵的操作，滥用正则表达式
* 使用了低效的算法

性能剖析会导致服务器变慢，但是为了定位一些性能平静下来，还是值得的，可以通过采样的方式发现严重问题。
### 关于MySQL的查询
#### 整体负载
**慢查询日志**
记录执行时间超过某一值的语句。是开销最低、精度最高的测量查询时间的工具。
分析慢查询日志时，先使用pt-query-digest生成一个剖析报告，它能够提供日志的一个整体的概览，接下来再去定为精确所在之处也不迟。
![pt-query-digest](https://i.loli.net/2020/05/28/eIOhlKNU8JgpwX5.png)

#### 单条查询优化
* `show profile`：服务器上执行的所有语句都会测量器耗费的时间
* `show status`：返回全局计数器，也有某个会话级别的计数器。显示某些活动的频繁程度。
* 慢查询日志
* `performance schema`：5.5之后会变得很强大，待考证。

#### 如何诊断间歇性问题
使用`show global status`/`show process list`/`查询日志`确定是单条查询还是服务器问题

## schema与数据类型的优化
### 优化的数据类型
#### 原则
* 更小的通常更好，但要确保没有低估值的范围
* 简单就好，整型比字符操作代价更低
* 尽量避免NULL值。但通常把可为NULL的列改为NOT NULL对性能提升不是很大，如果计划对该列上建立索引，那么最好不要设置NULL

在选择数据类型是，第一步确定合适的大类型（数字、字符串、时间等），然后选择具体的类型。

#### 整数类型
`TINYINT`、`SMALLINT`、`MEDIUMINT`、`INT`、`BIGINT`，也可以无符号，将上限提升一倍

分别是8,16,24,32,64为存储空间 $-2^{N-1}\leq x \leq 2^{N-1}$

#### 实数类型

`DECIMAL`类型可以指定精度，对于`DECIMAL`列可以指定小数点前后所允许的最大位数。`DECIMAL`(18,9)小数点两边各存储9个数字，一共使用9个字节，一个小数点，4个整数部分、4个小数部分。

`FLOAT`使用4个字节，`DOUBLE`使用8个字节。MySQL使用`DOUBLE`作为内部浮点计算的类型。

#### 字符串类型

`VARCHAR`、`CHAR`实现形式与数据库引擎有关。

`VARCHAR`用于存储可变长字符串，比定长更加节省空间，但update时可能需要做额外开销。`VARCHAR`需要使用1-2个额外字节记录字符的长度。

合适的情况：
* 列最大长度比平均长度大很多
* 列更新很少
* 使用了复杂的字符集UTF-8

`CHAR`适合存储很短的字符串，或者所有值接近同一个长度，并且删除所有的末尾空格。例如MD5值，因为定长的MD5值不容易产生碎片。

`BLOB`和`TEXT`，BLOB二进制、TEXT字符形式，BLOB和SMALLBLOB同义词，TEXT和SMALLTEXT同义词。当他们太大时，InnoDB会使用专门的存储区域来存储，在行内需要1-4个字节存储一个指针。

需要尽量避免使用这两种类型。

**使用枚举代替字符串**
* 存储非常紧凑，会根据列表值的数量压缩到1-2个字节中。将每个值在列表中的位置保存为整数，并且在frm的文件中保存数字-字符串映射关系的查找表。
* 内部排序按照枚举的存储顺序排序的。
* 字符列表是固定的，添加、删除都是需要使用`ALTER TABLE`的

#### 日期和时间类型

`DATETIME`：1001-9999精度为秒，YYYYMMDDHHMMSS

`TIMESTAMP`：1970-2038 秒数，比datetime的效率更高。

#### 位数据类型
`BIT`：存储一个或者多个True False的值（0，1的字符串）

如果存储一个`b'00111001'`的值，数字上下文的场景中，会导致返回57，而不是ascii码为57的字符‘9’，因此要谨慎使用BIT。
![屏幕快照 2020-05-30 下午4.59.25.png](https://i.loli.net/2020/05/30/l9qKUinm5ypakY6.png)

`SET`：一种代价相对来说较高的表示位的形式，一些列打包的bit位。但是改变列的代价相对较高，且无法在set列上通过索引查找。

#### 如何选定标识符

整数通常是最好的选择 很快，并且可以使用auto_increment

enum和set糟糕的选择

尽量避免使用字符作为标识列，当使用uuid值时，最好把“-”删除，按照数字的方式存储，还是不如递增的整数好用

一些ORM系统是常见的性能噩梦，存储的类型比较任意。

### schema设计中的陷阱

* 太多的列：行缓冲中将编码过的列转换成行数据结构的操作代价是非常高的
* 太多的关联：单个查询最好在12个表以内做关联
* 全能的枚举：0，1，2，3
* 变相的枚举
* 非此发明的NULL：不要刻意不用NULL

### 范式和反范式

范式化的数据库中，每个事实数据都会出现并且仅出现一次。
反范式化的数据库中，信息是冗余的可能存储在多个地方。

#### 范式的优点和缺点
优点：

* 更新操作比反范式操作快
* 当数据较好的范式化时，就只有很少或者没有重复数据，所以执行操作会更快
* 范式化的表通常更小，可以更好的放在内存里，执行操作会更快
* 很少有多余的数据意味着检索列表时更少需要DISTINCT或者GROUP BY
  
缺点：
通常需要关联，代价昂贵，使得一些索引策略无效

#### 反范式的优点和缺点
优点：
* 避免关联
* 进行更有效的索引策略

#### 实际应用中经常是混用范式化和反范式化
最常见的反范式化数据的方法是复制或者缓存，在不同的表中存储相同的特定列，可以使用触发器更新缓存值。

### 缓存表和汇总表
缓存表表示存储哪些可以比较简答的从schema中其他表中获取数据的表（获取的速度比较慢）
汇总表表示的是使用GROUP BY语句聚合数据的表

有时候可以在缓存表上使用别的数据库引擎，例如支持更小索引占用空间、全文搜索的MyISAM。

使用缓存表和汇总表时，要决定是定时维护还是定期重建数据。

重建汇总表和缓存表时，通常需要保证数据在操作时仍然可用，通过‘影子表’实现。

更快的读，更慢的写。

### 如何加快ALTER TABLE
有一些情况不需要重建表
* 移除一个列的AUTO_INCREMENT
* 增加、删除和更改enum set常量

技术是为想要的表结构创建一个新的.frm：
1. 创建一张有相同结构的空表，进行所需要的修改
2. FLUSH TABLES WITH READ LOCK，关闭所有表。
3. 交换frm文件
4. UNLOCK TABLES




