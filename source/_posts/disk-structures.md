---
title: MySQL存储架构
date: 2024-1-12 10:00:00
tags:
- MySQL
categories: MySQL
---

>
>
> 之前看<<从根儿上理解MySQL>>，对MySQL的数据存储架构有了一定的理解。但是时间久了还是容易生疏，目前趁机进行梳理，更加细节的对MySQL的存储结构进行总结，进而来巩固自己的知识点

#### MySQL的逻辑存储结构

> 大家都知道InnoDB进行数据管理的最小单位是页，MySQL进行读写都是通过页为基本的单位。那么这些页是如何被管理起来的呢？下面咱们就一起来看一下吧

MySQL各个版本的架构图是不一样的，下面这张图是MySQL 8.0的存储架构图，示意图1-1如下：

![InnoDB architecture diagram showing in-memory and on-disk structures. In-memory structures include the buffer pool, adaptive hash index, change buffer, and log buffer. On-disk structures include tablespaces, redo logs, and doublewrite buffer files.](https://dev.mysql.com/doc/refman/8.0/en/images/innodb-architecture-8-0.png)



InnoDB的数据页被逻辑存放在一个空间内，称为表空间（tablespace）。该表空间由段（segment）、区（extent）、页（page）组成。

在存储层面，MySQL表就是.frm和.ibd文件组成。在物理层面，MySQL表就是一个二进制文件。在二进制文件的基础上，分析其内部逻辑组织结构，示意图1-2如下：

![WX20240117-094145@2x.png](https://img1.imgtp.com/2024/01/17/v52DKTjh.png)

InnoDB存储引擎的逻辑存储结构，示意图1-3如下：

![WX20240117-094208@2x.png](https://img1.imgtp.com/2024/01/17/Z6Bt6TtO.png)

#### 表空间

表空间是一个物理容器，表空间存储的对象是段，在一个表空间中可以看成是一个或多个段。MySQL数据库有多个表空间，如系统表空间、用户表空间、临时表空间等。通过上面的逻辑结构示意图可以看到，表空间是InnoDB存储引擎的逻辑结构最高层，所有的数据都存放在表空间。

- 系统表空间（System Tablespace）

  有的文档又称为共享表空间，默认情况下，InnoDB会在数据目录创建一个名为ibdata的文件，大小为12MB，这个文件就是系统表空间在操作系统中的实体表示。这个文件是自扩展文件，当空间不够用的时候会自己增加文件的大小。在MySQL服务器，默认的系统表空间只有一份。另外，通过示意图1-1可知，系统表空间包含改变缓冲（change buffer）, 在低版本中还有Doublewrite buffer等。

  其中，Change Buffer是缓存那些不在buffer pool里的辅助索引的变化的特殊数据结构。在磁盘上，Change buffer是System Tablespaces的一部分，当数据库宕机时，索引的变更会被缓冲到磁盘的Change Buffer区域。

  这里的change buffer和buffer pool中的写缓冲池（change buffer）有一定的区别。

- 独立表空间（File-per-table Tablespace）

  在MySQL的5.6.6版本之后，InnoDB不会默认把数据表存储到系统表空间中，而是为每一个表建立一个独立的表空间。每创建一个表，会在操作系统的文件目录下创建一个文件名为表名.idb的文件。

  独立表空间的结构比较复杂，具体可参考掘金小册，示意图1-4如下：

  ![image_1crjo0hl4q8u1dkdofe187b10fa9.png-105.2kB](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/5/1/16a739f33df9307a~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.awebp)



其他表空间

- 撤销表空间 （Undo log Tablespace）：专门用来存放undolog 的日志文件
- 临时表空间（Temporary Tablespace）: 用来存放用户创建的临时表和磁盘内部临时表
- 通用表空间（General Tablespace）：通用表空间是共享表空间，可以存放多个表的数据

#### 段

> 段不是表空间中的物理空间，而是一个逻辑概念。由若干个零碎的页面（碎片区）和一系列完整的区组成。

如图1-3所示，表空间中有多种段组成，常见的有数据段即叶节点段（leaf node segment）、索引段即非叶子节点段（non-leaf node segment）、回滚段（rollback segment）等。

- 数据段：管理叶子节点的数据

- 索引段：管理非叶子节点的数据

- 回滚段：保存undo log，用于事务回滚

每向表中插入一条数据，本质上就是向该表的聚簇索引和所有的二级索引代表的B+树节点插入对应的数据。

InnoDB 对 B+ 树的叶子节点和非叶子节点进行了区别对待，也就是说叶子节点有自己独有的区，非叶子节点也有自己独有的区。存放叶子节点的区的集合就算是一个段，存放非叶子节点的区的集合也算是一个段。也就是说一个索引会生成两个段，一个叶子节点段和一个非叶子节点段。

InnoDB为每一个segment定义了一个INode Entry结构来记录段中的属性， INode Entry的结构，示意图1-5如下：

![WX20240117-094240@2x.png](https://img1.imgtp.com/2024/01/17/R0KEali4.png)

- Segment ID: 段的唯一标识ID
- NOT_FULL_N_USED：指在NOT_FULL链表中已经使用了多少页面
- 3个List Bae Node：分别为段的 FREE 链表、NOT_FULL 链表、FULL 链表定义的头节点
- Magic Number：魔法数用来标记INode Entry是否已经被初始化
- Fragment Array Entry：是一些零散的页面和一些完整的区的集合，该结构占 4 个字节，表示一个零散页面的页号

段链表

InnoDB为每个段中的区对应的XDS Entry结构建立了三个链表：

FREE链表：同一个段中，所有页面都是空闲的区对应的 XDES Entry 结构会被加入到这个链表。注意和直属于表空间的FREE 链表区别开，此处的 FREE 链表是附属于某个段的

NOT_FULL链表: 同一个段中，仍有空闲空间对应的XDS Entry 结构会被加入到这个链表

NOT_FULL链表: 同一个段中，没有空闲空间对应的XDS Entry 结构会被加入到这个链表

#### 区

因为段只是一个逻辑概念，不对应物理区域，因此，表空间实际可以看成是由若干个区组成的，每256个区被划分成一组。每个区包含64个页，因此，每个区的默认大小是 64 * 16B = 1M

对于每个区的页在物理空间是连续的，因此对于一个区的页可以减少随机I/O的次数，提高读取速度。

通过上面的示意图1-4可看出区的结构

其中，extent0的最开始3个页面的类型是固定的，分别是：

- FSP_HDR：登记整个表空间的一些整体属性以及本组所有的区。整个表空间只有一个 FSP_HDR 类型的页面。
- IBUF_BITMAP：存储本组所有的区的所有页面关于 INSERT BUFFER 的信息。
- INODE：存储了许多称为 INODE 的数据结构

其它extent的最开始的2个页面的类型也是固定的，分别是：

- XDES：全称是Extent Descriptor Entry，用来登记本组各个区的属性，是一种用来管理区的结构，下面会详细讲
- IBUF_BITMAP：同上

> XDES：为了方便管理区，MySQL 设计了 XDES Entry结构，每个区都对应着一个 XDES Entry，这个结构记录了对应的区的一些属性

XDES Entry的结构，示意图1-6如下：

![WX20240117-094300@2x.png](https://img1.imgtp.com/2024/01/17/ifdxjmsJ.png)

- Segment ID: 该区所在的段ID

- ListNode：这个部分可以将若干个 XDES Entry 结构串联成一个双向链表

- State: 区的状态，总共有四种类型，分别是：

  > FREE，空闲的区：现在还没有用到这个区中的任何页面；
  >
  > FREE_FRAG，有剩余空间的碎片区：表示碎片区中还有可用的页面；
  >
  > FULL_FRAG，没有剩余空间的碎片区：表示碎片区中的所有页面都被使用，没有空闲页面；
  >
  > FSEG，附属于某个段的区：每一个索引都可以分为叶子节点段和非叶子节点段，除此之外 InnoDB 还会另外定义一些特殊作用的段，在这些段中的数据量很大时将使用区来作为基本的分配单位。

- Page State BitMap: 这个部分共占用 16 字节，也就是 128 位，一个区默认有 64 个页，这 128 位被划分为 64 个部分，每个部分有 2 位，对应区中的 1 个页。第一个位表示对应的页是否是空闲的，第二个比特位还没有用

区链表

和段链表类似，区链表同样有三种类型

- 把状态为 FREE 的区对应的 XDES Entry 结构通过 List Node 来连接成一个链表，即 FREE 链表；

- 把状态为 FREE_FRAG 的区对应的 XDES Entry 结构通过 List Node 来连接成一个链表，即 FREE_FRAG 链表。

- 把状态为 FULL_FRAG 的区对应的 XDES Entry 结构通过 List Node 来连接成一个链表，即 FULL_FRAG 链表

碎片区：

> 段是以区为单位申请存储空间的，一个区默认占用 1MB 的空间，如果为段中很小的数据量申请区，则会造成大量空间的浪费，所以 InnoDB 提出了碎片区的概念，碎片区中的页并不都是为了存储同一个段中的数据而存在的，比如有的页用于段 A，有的用于段 B，有的页甚至哪个段也不属于。**碎片区直属于表空间，并不属于任何一个段。**

为段分配存储空间的策略是：

1. 起初向表中插入数据的时候，段是**从某个碎片区以单个页面**来分配存储空间的
2. 当某个段已经占用了 32 个碎片区页面（半个区）之后，就会申请以**完整的区**为单位来分配存储空间

#### 数据页

> 数据记录是按照行进行存储的，但是InnoDB的数据读取是按照页进行的，即数据库I/O的最小单位是页

上面讲到InnoDB进行数据读取的基本单位是页，这些页是通过区的ListNode双向链表结构链接起来的。下面看下数据页的结构具体是怎样的呢？

InnoDB数据页结构，示意图1-7如下：

![WX20240117-094318@2x.png](https://img1.imgtp.com/2024/01/17/WryNd9AN.png)

| 英文名称           | 中文名称         | 说明                                                         |
| ------------------ | :--------------- | ------------------------------------------------------------ |
| Filer Header       | 文件头           | 表示页的信息，其中Filer Header中有两个指针，分别指向前一个数据页和后一个数据页，是一个双向链表 |
| Page Header        | 页头             | 表示页的状态信息                                             |
| Infimum + supremum | 最小和最大的记录 | 两个虚伪的伪记录，分别表示页中最大和最小的记录               |
| User Records       | 用户记录         | 存储行记录内容                                               |
| Free Space         | 空闲空间         | 页中还没被使用的空间                                         |
| Page Directory     | 页记录           | 存储用户记录的相对位置，对记录起到索引作用                   |
| File Trailer       | 文件尾           | 校验页是否完整                                               |

> 数据页中的记录（User Records）按照主键顺序组成单向链表
>
> 数据页中的页目录（Page Directory）通过槽（Slot）将记录进行管理

通过下面的Table，看一下User Records是如何被InnoDB管理起来的

```mysql
mysql> CREATE TABLE page_demo(
    ->     c1 BIGINT(20) NOT NULL DEFAULT '0' AUTO_INCREMENT COMMENT '主键',
    ->     c2 BIGINT(20) NOT NULL DEFAULT '0',
    ->     c3 VARCHAR(255) NOT NULL DEFAULT '',
    ->     PRIMARY KEY (c1)
    -> ) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=Compact COMMENT='数据页demo';
Query OK, 0 rows affected (0.03 sec)
```

假设page_demo中有6条记录，InnoDB会将这些数据分成两组，这里并不是平均分，第一组只有一个最小记录，第二组是剩余的5条记录。示意图1-7如下：（关于记录头信息每一项的含义，请继续往下进行阅读）

![WX20240117-094336@2x.png](https://img1.imgtp.com/2024/01/17/ZMr2AoCf.png)

每个记录组的最后一条记录就是组内最大的那条记录，并且同一个组的最后一条记录的头信息中会存储该组一共有多少条记录，作为 n_owned 字段，其他记录的n_owned字段为0。

页目录（Page Directory）用来存储每组最后一条记录的地址偏移量，这些地址偏移量会按照先后顺序起来，每组的地址偏移量也被称之为槽（slot），**每个槽相当于指针指向了不同组的最后一个记录**。

大家可能有疑问的是，为什么上面进行分组的时候，槽0为什么只有一条记录？

> InnoDB 对每个分组中的记录条数都是有规定的，槽内的记录就只有几条：
>
> - 第一个分组中的记录只能有 1 条记录；
> - 最后一个分组中的记录条数范围只能在 1-8 条之间；
> - 剩下的分组中记录条数范围只能在 4-8 条之间。

另外需要思考的是，有了这些数据记录MySQL究竟是通过什么方法找到记录的呢？

> 其实就是二分法，因为每个槽中对应响应分组中的主键值最大的记录，可以通过最大主键值和目标主键进行比较，寻找到对应的槽，然后进行链表的单向寻找
>
> 假设现在要查询主键值为m的记录，最低槽 low = 0, 最高槽high=n，那么具体过程为：
>
> 1. 计算中间槽的位置 (low + high) / 2 = i, 槽i对应的主键值k和m进行比较，如果k < m，设置low = i + 1；否则设置 high = i - 1
> 2. 重新计算中间槽的位置  (low + high) / 2 = i, 不断进行比较
> 3. 如果high - low = 1，那么表示目标主键值在槽high中，那么槽high中根据next_record进行遍历查找。但是这里需要知道槽high中的初始指针，因为每个槽都是相连的，因此槽low的主键值对应记录的下一条记录就是槽high的第一个记录

所以在一个数据页中查找指定主键值的记录的过程分为两步：

1. 通过二分法确定该记录所在的槽，并找到该槽所在分组中主键值最小的那条记录。
2. 通过记录的`next_record`属性遍历该槽所在的组中的各个记录。

#### 数据行

MySQL 有四种行格式：Redundant，Compact，Dynamic，Compressed。我们可以在创建表的时候指定字符集和行格式。

MySQL 5.0之后默认使用的是 Dynamic 的行格式

因为Redundant是MySQL 5.0之前的格式，现在很少使用，不进行讲解，如果感兴趣可以从网上搜素资料来详细了解。下面主要讲解Dynamic行格式和Compact行格式

Compact行格式，示意图1-8如下：

![WX20240117-094518@2x.png](https://img1.imgtp.com/2024/01/17/RmNOrwwV.png)

Compact行格式描述信息说明：

| 名称             | 大小(Bit) | 描述                                                         |
| ---------------- | --------- | ------------------------------------------------------------ |
| 变长字段长度列表 | 不定      | 存储的是每个非空的变长字段的数据长度，同样也是逆序存放的。如果表中没有变长的数据类型的话，就不需要该列表了。如果长度小于 255B，就用 1B 表示，否则用 2B 表示，最大不会超过 2B，因此 MySQL 中变长类型的最大字节长度限制为 65535。 |
| NULL值列表       | 1byte     | 如果表中没有允许 NULL 的列，那么 NULL 标记位就不存在了。否则将每个允许存储 NULL 的列对应一个二进制位并且逆序存放，然后将允许为 NULL 的位标为 1，不允许的标为 0；且必须用整数个字节的位表示，否则在高位补 0。 |
| 记录头信息       | 5byte     | 参考下面                                                     |

Compact 格式的记录头具体信息说明：

| 名称         | 大小（bit） | 描述                                                         |
| ------------ | ----------- | ------------------------------------------------------------ |
| 预留位1      | 1           | 未使用                                                       |
| 预留位2      | 1           | 未使用                                                       |
| delete_mask  | 1           | 该记录是否删除，0：未删除 1：删除                            |
| min_rec_mask |             | B+树每层非叶子节点小记录都会添加该标记                       |
| n_owned      | 4           | 当前记录组拥有记录数                                         |
| heap_no      | 13          | 当前记录在页面堆位置信息                                     |
| record_type  |             | 表⽰当前记录的类型，0表⽰普通记录，1表⽰B+树⾮叶⼦节点记录,2表⽰⼩记录，3表⽰⼤记录 |
| next_record  | 16          | 下一条记录的相对位置                                         |



在讲Dynamic行格式之前先讲一下行溢出问题：

> 当列的类型为 VARCHAR、 VARBINARY、 BLOB、TEXT 时，该列超过 768byte 的数据溢出存放到溢出页中，然后通过一个偏移量指针将两者关联起来，这就是行溢出机制。这样可以有效地防止单个列太大导致单个数据页中存放的行记录过少而让 IO 飙升的情况。

行溢出示意图1-9如下：

![image_1d48e3imu1vcp5rsh8cg0b1o169.png-149kB](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/3/12/169710e9aab47ea5~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.awebp)

Dynamic行格式：

Dynamic 和 Compact 行格式类似，只是在处理行溢出数据时有区别。Dynamic 的数据页不会存储真实数据的前 768 字节，而是存储 20 个字节的指针来指向溢出页，即将所有的溢出数据都存在溢出页中。

关于其他的数据行格式，这里就不再赘述了，详细的可以参考掘金小册中的相关章节。

#### 总结

本文介绍了 MySQL 底层的逻辑存储结构，每一条数据都是一个数据行，数据行是存储数据的基本单位。若干条数据会组成一个数据页，数据页是读写的基本单位，可以减少磁盘 IO 的次数。若干个连续的页会组成一个区，可以降低随机 IO 的次数，来提高查询效率，同时也便于进行管理。不同的区可以按照逻辑分布到不同的段中，而段只是一个逻辑概念，真正进行存储空间分配是按照区进行的，通过这种方式可以更好地进行数据管理以及提高空间利用率。对于整体MySQL存储体系架构图的形成，可以不断的理解上面的图示，进行记忆的加深。附录里面的图很有帮助，如果对图示中的每一个环节都明白了，那么对MySQL的存储架构的知识树也会在脑海中形成了。

#### 附录

下面这张MySQL表空间整体结构图画的非常详细，示意图1-10如下：

![image_1d9ppsbelendcbb13hghhn18pe9.png-3564.2kB](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/5/1/16a739f4a99c9a08~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.awebp)



#### 参考文档

[1] https://www.zhihu.com/question/26398102

[2] https://juejin.cn/book/6844733769996304392/section/6844733770050830344

[3] https://blog.csdn.net/weixin_43733154/article/details/106913924




