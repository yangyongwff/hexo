---
title: MySQL日志介绍
date: 2024-06-01 23:20:31
tags:
---

## 前言

MySQL日志主要可以细分为错误日志、查询日志、慢查询日志、二进制日志、事务日志（重做日志和回滚日志)和中继日志等几类。本篇文章主要针对这几种日志进行分析，对每种日志的产生阶段和作用进行介绍，另外从整体上看清MySQL日志之间的关联。

## 错误日志

错误日志包含了有关服务器运行时发生的各种错误和警告的信息。通常在MSQL的配置文件中指定。常见的配置文件包括my.cnf或my.ini,具体取决于操作系统。查看错误日志对我们解决日常问题是非常重要的。

查看错误日志文件的完整路径，通过执行以下SQL查询语句

> SHOW VARIABLES LIKE 'log_error'
>
> | Variable_name | Value     |
> | :------------ | :-------- |
> | log_error     | ./err.log |

## 查询日志

查询日志在MySQL中被称为general log(通用日志)，查询日志里的内容不要被"查询日志"误导，它其实记录了数据库执行的所有命令，并不仅仅是查询命令。这对于性能调整和故障排除非常有用。

查询日志的位置通常在MySQL的配置文件中指定。可以通过执行以下SQL查询语句来查找查询日志文件的位置
SHOW VARIABLES LIKE 'general_log_file'; 这将显示查询日志文件的完整路径。

启用查询日志
如果查询日志被禁用，可在MySQL配置文件中，找到并修改以下配置项：
general_log = 1
或者使用以下SQL语句在运行时启用：
SET GLOBAL general_log 1;

禁用查询日志
一旦你完成了查看查询日志的操作，可以选择将查询日志禁用，以避免日志文件变得庞大。
general_log = 0
或者使用以下SQL语句在运行时禁用：
SET GLOBAL general_log = 0
请注意，启用查询日志可能会对系统性能产生一定的影响，因此在使用完毕后及时禁用。在生产环
境中，最好只在需要排查问题时启用查询日志。

## 慢查询日志

慢查询会导致CPU，IOPS，内存消耗过高。当数据库遇到性能瓶颈时，大部分时间都是由于慢查询导致的。 开启慢查询日志，可以让MySQL记录下查询超过指定时间的语句，之后运维人员通过定位分析，能够很好的优化数据库性能。同样的，慢查询日志里的内容也记录了数据库执行的所有命令。

对慢查询日志的分析可以通过一些工具如pt-query-digest对已经生成的日志文件进行分析，或者执行show full processlist进行实时查看抓取的SQL日志，这是对MySQL链接执行的现场快照。

执行show full processlist后，各字段的含义如下：

| 字段    | 含义                                                         | 作用                                                         |
| ------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| id      | mysql连接的唯一标识                                          | 操作或过滤特定的连接，比如使用kill命令时。定位问题连接，比如查看事务、锁等，其中会有一个thread_id, 该线程会对应processlist.id;通过这个关系在定位复杂问题时，可以快速的定位问题 |
| user    | 创建数据库连接的用户                                         | 可用于用户行为追踪和用户行为统计                             |
| host    | 创建连接的服务器，一般由服务器IP+端口号组成                  | 统计请求来源                                                 |
| db      | 该连接执行SQL的数据库                                        |                                                              |
| command | 表示连接状态 常见的有 Sleep(睡眠) Query（查询） Connect(连接) |                                                              |
| state   | SQL执行的状态                                                | 结合time使用，持续的时间比较长，则有问题的概率较大           |
| time    | 连接在当前状态的持续时间                                     | 判断SQL是否正常的关键要素                                    |
| info    | 正在执行的完整的SQL语句                                      | 定位到问题SQL                                                |

查询是否开启慢查询日志，可以通过下面的参数进行查看：

show variables like '%query%'

| Variable_name                | Value    |
| ---------------------------- | -------- |
| binlog_rows_query_log_events | on       |
| ft_query_expansion_limit     | 20       |
| have_query_cache             | YES      |
| long_query_time              | 0.100000 |
| query_alloc_block_size       | 8192     |
| query_cache_limit            | 0        |
| query_cache_min_res_unit     | 4096     |
| query_cache_size             | 0        |
| query_cache_type             | OFF      |
| query_cache_wlock_invalidate | OFF      |
| query_prealloc_size          | 8192     |
| slow_query_log               | ON       |
| slow_query_log_file          | slow.log |
|                              |          |

其中，slow_query_log = on # 启用

slow_query_log_file=slow.log #设置慢查询日志的位置

long_query_time = 0.100000 设置阈值，单位是秒

## 事务日志

事务日志包括重做日志（redo log）和回滚日志（undo log）两部分，是数据备份和恢复的关键。下面先简单的看一下SQL执行的全过程。

![未命名文件 _5_.png](https://img2.imgtp.com/2024/05/05/NVgvfozN.png)

### 回滚日志（undo log）

#### undo log的内容和作用

**内容：**

Undo Log记录的是逻辑日志，也就是SQL语句。

比如：当我们执行一条insert语句时，Undo Log就记录一条相反的delete语句。

**作用：**

1. 回滚事务时，恢复到修改前的数据。
2. 实现 **MVCC（多版本并发控制，Multi-Version Concurrency Control）** 。

MySQL事务中原子性就是使用**Undo Log**实现的。

#### undo log如何回滚到上一个版本

每一个数据行都有两个隐藏列，其中一个是trx_id（最近一次提交的事务ID）和roll_pinter（上个版本的地址），建立一个版本链。并在事务读取的时候生成一个ReadView视图，在不同的隔离级别下生成不同的视图。

关于事务的隔离级别在之后的文章中详细介绍，这一块还是非常重要的知识。

### 重做日志（redo log）

#### redo log的内容和作用

**内容：**

redo log记录的是物理日志，也就是磁盘数据页的修改。

**作用：**

保证在服务崩溃后，仍能够把事务中变更的数据持久化到磁盘中。

MySQL事务中的持久性就是**Redo Log**实现的

#### redo log的写入时机

通过上面的SQL执行过程图可以看到，在事务提交之前，在步骤6中会先将待修改的数据同步到redo log buffer中，之后会根据不同的刷新策略，刷新到redo log file。

这里就会带来一个问题，为什么会有写入redo log buffer的过程，为什么不是一步到位，直接写入到redo log file？

主要是因为直接写入磁盘会有严重的性能问题

> 1. 前面讲过InnoDB 的基本存储单元是数据页，可能本次的修改只涉及页中的几个字节，但是却需要刷新整个数据页，对预计整个资源来讲是极大的浪费
> 2. 一个事务可能修改了几个数据页，而数据页之间不是连续的，就会产生随机IO，性能更差

这种方式就是WAL(Write Ahead Logging)，就是预写日志，先写日志，再写磁盘

#### redo log的刷盘策略

写入redo log buffer后，并不会立刻持久化到redo log file，需要等待操作系统调用fsync()操作，才会刷新到磁盘。

具体什么时候可以把redo log buffer刷到redo log filee**中，可以通过**innodb_flush_log_at_trx_commit**参数配置决定。

| 参数值        | 含义                                                         |
| ------------- | ------------------------------------------------------------ |
| 0（延迟写）   | 提交事务后，不会立刻刷新到OS Buffer，而是等待1s后刷新到OS Buffe，并调用fsync()，写入redo log file,中间可能会有1s的数据丢失风险 |
| 1（实时写）   | 每次提交事务后，都会立刻刷新到OS Buffer，并调用fsync()，写入redo log file，性能较差 |
| 2（延迟刷新） | 每次提交的事务只刷新到OS Buffer，1s后再调用fsync()刷新到redo log file |

InnoDB的Redo log file是固定的大小，可以配置为每组4个文件，每个文件的大小是1G，那么redo log file可以记录4G的数据

它采用的是循环写入覆盖的方式，从头开始写，写到末尾又从头开始写，如下图所示。

![未命名文件 _5_.png]( https://img-hxy021.didistatic.com/static/iportal/upload/1670e378a86ad6d424b99e0788f2f95d.png)



每个日志文件组有两个重要的属性，check point和write pos

check point是记录当前要擦除的位置，也是往后推移

write pos 是当前记录的位置，也是一边写一边往后推移

每次刷盘redo log记录到日志文件组中，write pos位置就会后移更新

每次MySQL加载日志文件组恢复数据时，会清空加载过的redo log记录，并把check point后移更新。

write pos和check point之间空着的部分可以用来写入新的redo log记录

如果write pos追上了check point 代表日志文件组满了，这时候不能再写入新的redo log日志记录了，MySQL得停下来，清空一些记录，把check point推进一下。

## BinLog二进制日志

binlog日志按顺序记录了所有更新数据库的语句（如DDL和DML）并以二进制的形式保存在磁盘中。但是不包含任何没有修改数据的语句（如select,show等）。只要发生了表数据更新，都产生binlog日志。

主要作用：数据备份和主从同步

binlog的记录格式有三种形式：statement、row和mixed，对比如下：

| 格式      | 含义                                                         | 优点                                                         | 缺点                                                         |
| --------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| statement | 基于SQL语句的复制，记录的是更新数据库的SQL语句，这些语句同步时，会被其他节点执行。 | 不需要记录每一行的变化，节省了binlog日志量。节约了IO，提高了性能 | 由于记录的只是执行语句，为了这些语句能在slave正确执行，还需要记录一些这些语句在执行时候的原始信息，从而保证slave和master的执行语句可以得到相同的执行结果。比如update_time=now(),这里会获得当前的系统的当前时间，直接执行会导致与原库的数据不一致 |
| row       | 5.1.5版本的MySQL才开始支持row level的复制，它不记录sq语句上下文相关信息，仅保存哪条记录被修改。 | binlog中可以不记录执行的sql语句的上下文相关的信息，仅需要记录那一条记录被修改成什么了。所以row的日志内容会非常清楚的记录下每一行数据修改的细节。而且不会出现某些特定情况下的存储过程，或function,以及trigger的调用和触发无法被正确复制的问题。 | 每条数据的更改被详细记录，如整表删除，alter表等操作涉及的数据行都会记录，ROW格式会产生大量日志，比较占用空间，恢复与同步时会更消耗IO资源，影响执行速度 |
| mixed     | 5.1.8版本开始，以上两种格式的混合版，对于DDL只对SQL语句进行记录，对DML操作则会进行判断，如果判断会造成主从不一致，就会采用row格式记录，反之则用statement格式记录。 | 既节省了空间，又提高数据库性能，保证数据同步的一致性         | 无法对误操作数据进行单独恢复                                 |

**什么时候写入binlog？**

**Bin Log**采用追加写入的模式，并不会覆盖原有日志，所以可以用来恢复到之前某个时刻的数据。

binlog 区别于 redo log，每个事务所在的线程都会存在一个 binlog cache，并且只有事务提交时会持久化到磁盘文件（binlog file）。

**binlog什么时候刷盘？**

binlog的刷新策略可以**sync_binlog**配置参数指定。

| 参数值      | 含义                                                         |
| ----------- | ------------------------------------------------------------ |
| 0（延迟写） | 每次提交事务都不会刷盘，由系统自己决定什么时候刷盘，可能会丢失数据。 |
| 1（实时写） | 每次提交事务，都会刷盘，性能较差。                           |
| N（延迟写） | 提交N个事务后，才会刷盘。                                    |



## 参考文档

[1] https://segmentfault.com/a/1190000042041728

[2] https://blog.csdn.net/qq_42435377/article/details/124253898



