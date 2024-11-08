---
title: 定时任务系统原理与实现分析
date: 2024-3-20 20:37:51
tags:
- 定时任务原理
categories: 中间件
---

# 一、定时任务系统的实现方式

定时任务核心接口：

| 接口名称     | 作用                                                     |
| ------------ | -------------------------------------------------------- |
| tick         | 不同实现方式表示含义不同                                 |
| addTimer     | 向容器中添加一个定时器，并返回定时器的指针               |
| delTimer     | 根据传入的定时器指针删除容器中的一个定时器，并且销毁资源 |
| resetTimer   | 重置一个定时器                                           |
| getMinExpire | 不同实现方式表示含义不同                                 |



各种实现方式对比：

| **实现方式**   | **AddTimer** | **DelTimer**      | **ToDoTimer**                                                |
| -------------- | ------------ | ----------------- | ------------------------------------------------------------ |
| 基于链表       | O(1)         | O(n)              | O(n)                                                         |
| 基于排序链表   | O(n)         | O(1)              | O(1)                                                         |
| **基于最小堆** | O(lgn)       | O(1) （后面分析） | O(1)                                                         |
| **基于时间轮** | O(1)         | O(1)              | O(1)(前提是使用多个轮子来实现，不过单个轮子也要比O(n)快得多) |

## 1.1 最小堆

> 最小堆是一种特殊的完全二叉树，任意非叶子节点的值不大于其子节点

> 具体实现代表就是 JDK 中的 Timer 定时器

![img](https://cooper.didichuxing.com/uploader/f/oRdVuewrDV8h3ovE.png?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3MzEwNzc5ODMsImZpbGVHVUlEIjoieGQwQmpJdFFSWWtvWnFwdiIsImlhdCI6MTczMTA3NzM4MywiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwidXNlcklkIjoxMDAwMDAyODY0MH0.fJyBLlFIktnCpHrJvdDpmQ8u4UraIQ4QoHyJvmSJrwg)



**思考1：如何选取定时任务的键？**

> 通常情况下，最小堆的键是定时器任务的到期时间（即任务需要执行的时间点）。这是因为最小堆的数据结构特性使得堆顶元素总是具有最小的键值，在定时器场景中，这意味着堆顶元素是下一个需要执行的任务。

```
示例
假设有以下几个定时器任务：
任务A，到期时间为1622548800（Unix时间戳）
任务B，到期时间为1622548805
任务C，到期时间为1622548810


最小堆后的结构将如下：
       1622548800 (A)
       /        \
1622548805 (B)  1622548810 (C)
```



**思考2：生成最小堆的时候会把任务的所有键一次性全部插入？**

> 任务的所有键可以一次性加入堆中，或者逐个插入。选择哪种方法取决于具体应用场景和实现方式

| 插入方式       | 描述                                                         | 缺点                                                         | 优点                                                         | 适用范围                       |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------ |
| 一次性插入     | 将全部解析cron表达式，并一次性加入所有任务的键（到期时间）一次性加入堆中，然后通过堆化操作构建最小堆。 | 内存占用高：如果任务时间点数量巨大，会占用大量内存初始解析开销大：一次性解析所有时间点可能需要较长时间 | 简化调度逻辑：所有任务一次性加入堆，调度器只需要从堆中取出任务并执行减少解析频率：避免了频繁解析 cron 表达式 | 任务时间点数量有限或明确的情况 |
| **渐进式插入** | 每次只解析最近的一段时间，将这段时间内的任务加入到最小堆中，并定期解析接下来的时间段。 | 复杂调度逻辑：需要管理定期解析和任务加入的逻辑。解析频率高：需要频繁解析 cron 表达式。 | 内存占用低：每次只处理一小部分任务，内存占用较低解析开销分散：解析工作分散到多个时间点，减少单次解析的开销 | 任务时间点分布广泛或无限的情况 |



**思考3：**思考1和思考2解决了如何构建最小堆的问题，并未告诉什么时候扫描最小堆，以及如何扫描？

> 将所有定时器中超时时间最小的一个定时器的超时值作为心搏间隔，这样，一旦心搏函数tick被调用，超时时间最小的定时器必然到期，我们就可以在tick函数中处理该定时器。然后，再从剩余的定时器中找到超时时间最小的一个，并将这段最小时间设置为下一次心搏间隔，如此反复，就实现了较为精确的定时

重要函数：

tick ：在tick函数中循环查找超时值最小的定时器，并调用其回调函数，直到找到的定时器的没有超时，就结束循环

getMinExpire：获取容器中超时值最小的绝对时间



**关键函数时间复杂度分析**

> 主要借助哈希表进行优化查找过程

在基于最小堆的数据结构中，定时器删除操作（DelTimer）的时间复杂度之所以可以达到 O(1)，是因为某些具体实现中的优化手段。最小堆的删除操作通常涉及到以下步骤：

1. **找到要删除的元素**：在一般的最小堆（例如优先队列）中，这个操作需要 O(logn) 的时间复杂度，因为需要遍历整个堆来找到元素。
2. **删除元素并重构堆**：这一步通常是 O(log⁡n) 的时间复杂度，因为需要从堆中移除元素，然后进行堆的调整（堆化）。

某些定时器实现中，通过结合哈希表来优化这两个步骤，使得删除操作可以在 O(1)的时间复杂度内完成。优化思路如下：

1. **哈希表记录位置**：在插入元素到最小堆时，同时将元素的位置记录在一个哈希表中。
2. **快速定位元素**：当需要删除某个元素时，可以通过哈希表快速找到该元素在堆中的位置，这一步操作的时间复杂度是 O(1)
3. **交换并删除**：找到元素后，将其与堆的最后一个元素交换，然后移除最后一个元素。再对交换位置进行堆化操作。
4. **堆化调整**：这一步的时间复杂度是 O(log⁡n)



## 1.2 时间轮

Kafka、Dubbo、ZooKeeper、Netty、Caffeine 中都有对时间轮的实现



- Round时间轮

![img](https://cooper.didichuxing.com/uploader/f/YNiuwqvpqnSPx0Mz.png?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3MzEwNzc5ODMsImZpbGVHVUlEIjoieGQwQmpJdFFSWWtvWnFwdiIsImlhdCI6MTczMTA3NzM4MywiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwidXNlcklkIjoxMDAwMDAyODY0MH0.fJyBLlFIktnCpHrJvdDpmQ8u4UraIQ4QoHyJvmSJrwg)



轮中实线指针指向轮中的一个槽（slot）。它以恒定的速度顺时针转动，每转动一步就指向下一个槽（虚线指针指向的槽），每次转动一次称为一个滴答（tick）。一个滴答的时间称为时间轮的槽间隔si（slot interval），它实际上就是定时器的心搏时间。该时间轮共有N个槽，因此它转一圈所需的时间就是N*si。每个槽中保存了一个定时器链表，时间轮的结构与哈希链表的结构是比较相似的。槽中的每条链表上的定时器具有相同的特征：它们的定时相差N*si的整数倍。时间轮正是利用这个关系将定时器散列到不同的链表中的。



假如现在指针指向槽cs，我们要添加一个定时时间为ti的定时器，则该定时器将被插入槽ts对应链表中的：

*ts*=(*cs*+(*ti*/*si*))%*N*



**思考：这里ts只是指出了slot的位置，但是未包含圈数，怎么办呢？**

> 维护圈数变量或者多层次时间轮

>

- 分层时间轮

类似手表，Kafka 的延迟消息采用的就是这种方案

![img](https://cooper.didichuxing.com/uploader/f/x7gbo6RQco5gYNuh.png?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3MzEwNzc5ODMsImZpbGVHVUlEIjoieGQwQmpJdFFSWWtvWnFwdiIsImlhdCI6MTczMTA3NzM4MywiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwidXNlcklkIjoxMDAwMDAyODY0MH0.fJyBLlFIktnCpHrJvdDpmQ8u4UraIQ4QoHyJvmSJrwg)



上图的时间轮(ms -> s)，第 1 层的时间精度为 1 ，第 2 层的时间精度为 20 ，第 3 层的时间精度为 400。假如我们需要添加一个 350s 后执行的任务 A 的话（当前时间是 0s），这个任务会被放在第 2 层（因为第二层的时间跨度为 20*20=400>350）的第 350/20=17 个时间格子。

当第一层转了 17 圈之后，时间过去了 340s ，第 2 层的指针此时来到第 17 个时间格子。此时，第 2 层第 17 个格子的任务会被移动到第 1 层。

任务 A 当前是 10s 之后执行，因此它会被移动到第 1 层的第 10 个时间格子。

注意：任务会不断地计算并移动到不同层级的时间轮



**时间复杂度分析**

通过上面的分析可知，任务写入和执行的时间复杂度都是O(1)



# 二、Quartz分布式定时任务系统的基本实现原理

> 上面提到的定时任务的解决方案都是在单机下执行的，适用于比较简单的定时任务场景比如每天凌晨备份一次数据。如果要将最小堆或者时间轮应用在分布式环境下，还需要解决

> **任务分配**：将任务分配到不同的节点进行处理。

> **节点协调**：确保各节点协同工作，不重复调度任务。

> **任务持久化和恢复**：确保任务数据不会因为节点故障而丢失。

>

> 如需要一些高级特性比如支持任务在分布式场景下的分片和高可用的话，我们就需要用到分布式任务调度框架了，比如著名的Elastic-Job并未使用时间轮算法



XXX Job的任务调度功能一般都是基于quartz来开发的，并且Spring也集成了Quartz模块。 下面主要介绍Quartz的基本原理

GItHub: https://github.com/quartz-scheduler/quartz



## **2.1 Quartz的核心组件**

![img](https://cooper.didichuxing.com/uploader/f/vxmAkeHL4ISZRdo1.png?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3MzEwNzc5ODMsImZpbGVHVUlEIjoieGQwQmpJdFFSWWtvWnFwdiIsImlhdCI6MTczMTA3NzM4MywiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwidXNlcklkIjoxMDAwMDAyODY0MH0.fJyBLlFIktnCpHrJvdDpmQ8u4UraIQ4QoHyJvmSJrwg)



**Job任务**：表示被调度的任务

```
public interface Job {
    void execute(JobExecutionContext context)
        throws JobExecutionException;
}
```



**Trigger触发器**：Trigger 定义调度时间的元素，即按照什么时间规则去执行任务。一个Job可以被多个Trigger关联，但是一个Trigger 只能关联一个Job

支持的Trigger类型：SimpleTrigger、CronTrigger、DailyTimeIntervalTrigger、CalendarIntervalTrigger



**Scheduler****调度器** ：工厂类创建Scheduler，根据触发器定义的时间规则调度任务

```
// 创建调度器
Scheduler scheduler= StdSchedulerFactory.getDefaultScheduler();
...
// 将作业和触发器注册到调度器中
scheduler.scheduleJob(job, trigger1);
```



## 2.2 Quartz的核心表

```
qrtz_blob_triggers : 以Blob 类型存储的触发器。
qrtz_calendars：存放日历信息， quartz可配置一个日历来指定一个时间范围。
qrtz_cron_triggers：存放cron类型的触发器。
qrtz_fired_triggers：存放已触发的触发器。
qrtz_job_details：存放一个jobDetail信息。
qrtz_job_listeners：job**监听器**。
qrtz_locks： 存储程序的悲观锁的信息(假如使用了悲观锁)。
qrtz_paused_trigger_graps：存放暂停掉的触发器。
qrtz_scheduler_state：调度器状态。
qrtz_simple_triggers：简单触发器的信息。
qrtz_trigger_listeners：触发器监听器。
qrtz_triggers：触发器的基本信息
```



quartz通过quartz.properties配置文件，实现数据的存储和持久化

```
# 设置数据库驱动
org.quartz.jobStore.driverDelegateClass = org.quartz.impl.jdbcjobstore.StdJDBCDelegate
# 设置数据库连接URL
org.quartz.jobStore.dataSource = myDS
org.quartz.dataSource.myDS.driver = com.mysql.jdbc.Driver
org.quartz.dataSource.myDS.URL = jdbc:mysql://localhost:3306/quartzdb
org.quartz.dataSource.myDS.user = root
org.quartz.dataSource.myDS.password = root
org.quartz.dataSource.myDS.maxConnections = 5
```





## **2.3 Quartz的核心运行机制**

![img](https://cooper.didichuxing.com/uploader/f/PgvR1NlWxPnbNLA7.png?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3MzEwNzc5ODMsImZpbGVHVUlEIjoieGQwQmpJdFFSWWtvWnFwdiIsImlhdCI6MTczMTA3NzM4MywiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwidXNlcklkIjoxMDAwMDAyODY0MH0.fJyBLlFIktnCpHrJvdDpmQ8u4UraIQ4QoHyJvmSJrwg)



从上图可以看到，Quartz的核心流程大致分为三个阶段:

| 阶段   | 描述                       | 详细描述                                                     |
| ------ | -------------------------- | ------------------------------------------------------------ |
| 阶段一 | 获取调度实例阶段           | 通过getScheduler方法根据配置文件加载配置和初始化，创建线程池 ThreadPool(默认是SimpleThreadPool，用来执行Quartz调度任务)，创建调度器 QuartzScheduler，创建调度线程 QuartzSchedulerThread，并将调度线程初始状态设置为暂停状态 |
| 阶段二 | 绑定JobDetail和Trigger阶段 | 使用调度器将 JobDetail 和 Trigger 绑定在一起，并注册到调度器中 并持久化JobDetail和trigger 比如每一次编辑Cron表达式进行发布后，需要对 Trigger 进行更新，并触发一次持久化操作 |
| 阶段三 | 启动调度器阶段             | 参考下图                                                     |



阶段二核心代码：

```
StdScheduler.scheduleJob
   
public Date scheduleJob(JobDetail jobDetail, Trigger trigger)
        throws SchedulerException {
     		//这里实际调用的是QuartzScheduler
        return sched.scheduleJob(jobDetail, trigger);
}


public Date scheduleJob(JobDetail jobDetail,
            Trigger trigger) throws SchedulerException {
			//...省略...
        //持久化JobDetail和trigger
        resources.getJobStore().storeJobAndTrigger(jobDetail, trig);
      	//通知scheduler监听者
        notifySchedulerListenersJobAdded(jobDetail);
        notifySchedulerThread(trigger.getNextFireTime().getTime());
        notifySchedulerListenersSchduled(trigger);
        return ft;
    }
```



阶段三流程：

1. `Scheduler`会调用`QuartzScheduler`的`Start()`方法，这时候会把调度线程从暂停切为启动状态
2. 调用notifySchedulerListenersStarted，调度器监听器收到通知后，通知`QuartzSchedulerThread`正式执行
3. `QuartzSchedulerThread`会从`SimpleThreadPool`查看下有多少可用工作线程，然后找`JobStore`去拿下一批符合条件的待触发的`Trigger`任务列表，包装成`TriggerFiredBundle`。通过`JobRunShellFactory`创建`FiredTriggerBundle`的执行线程实例`JobRunShell`，
4. 然后把`JobRunShell`实例交给`SimpleThreadPool`的工作线程去执行。`SimpleThreadPool`会从可用线程队列拿出对应数量的线程，去调用`JobRunShell`的`run()`方法，此时会执行任务类的`execute`方法 : `job.execute(JobExecutionContext context)`。



**思考：Trigger数据的写入是否和前面介绍的也是渐进式的呢？**

> 每当任务调度器需要知道下一个触发时间时，它会基于当前时间动态地计算下一个触发时间。这意味着 Quartz 只会在需要时计算并缓存下一个触发时间，而不是提前计算和存储所有时间点。

> Quartz 会将 `Trigger` 的定义和状态信息存储到数据库中，这包括 Cron 表达式本身和触发器的元数据（例如下一个触发时间）。



**思考：****Quartz集群如何保证高并发下不重复跑？**

> Quartz节点之间是否存在分布式锁去控制？答案是肯定的。

> Quartz是通过数据库去作为分布式锁来控制多进程并发问题，Quartz加锁的地方很多，Quartz是使用悲观锁的方式进行加锁，让在各个instance操作Trigger任务期间串行

**锁的类型**：

- **TRIGGER_ACCESS**：用于控制触发器的调度访问，确保只有一个节点可以调度某个触发器。
- **JOB_ACCESS**：用于控制任务的执行访问，确保任务执行期间的原子性。
- **CALENDAR_ACCESS**：用于控制日历的访问，确保日历操作的原子性。
- **STATE_ACCESS**：用于控制调度器状态的访问，确保调度器状态变更的原子性。
- **MISFIRE_ACCESS**：用于控制错过触发时间的任务，确保错过的任务能正确处理。



加锁对任务执行的影响

虽然加锁是为了确保任务调度的原子性和一致性，但 Quartz 通过多种机制尽量减少锁对其他任务执行的影响

1. **细粒度的锁**：Quartz 使用多种锁类型，每种锁仅用于特定的操作，避免了全局锁的使用。这种细粒度的锁机制确保了大部分操作可以并行进行，而不互相阻塞。
2. **短时间持有锁**：Quartz 设计上尽量减少持有锁的时间。获取锁的操作尽可能快地完成，以便尽快释放锁，减少对其他操作的影响。
3. **线程池并行执行**：Quartz 使用线程池并行执行任务。即使某个任务获取锁进行调度，其他任务仍然可以通过线程池并行执行，不会因为某个锁而阻塞所有任务的执行。



**思考：****Quartz集群如何保证高并发下不漏跑？**

定时任务很容易出现漏跑的情况，如

- 服务重启，没能及时执行任务，就会misfire
- 工作线程去运行优先级更高的任务，就会misfire
- 任务的上一次运行还没结束，下一次触发时间到达，就会misfire



Misfire的大致流程：

![img](https://cooper.didichuxing.com/uploader/f/xt0ub75jHJjqnewc.png?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3MzEwNzc5ODMsImZpbGVHVUlEIjoieGQwQmpJdFFSWWtvWnFwdiIsImlhdCI6MTczMTA3NzM4MywiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwidXNlcklkIjoxMDAwMDAyODY0MH0.fJyBLlFIktnCpHrJvdDpmQ8u4UraIQ4QoHyJvmSJrwg)





MisfileHandler通过JobStoreSupport去查询有没有misfire的任务，查询条件是当前状态是waiting，下一次trigger时间< 当前时间-misfire预设阈值(默认1分钟)



```
int misfireCount = (getDoubleCheckLockMisfireHandler()) ?
                   getDelegate().countMisfiredTriggersInState(
                       conn, STATE_WAITING, getMisfireTime()) : 
                   Integer.MAX_VALUE;




    String COUNT_MISFIRED_TRIGGERS_IN_STATE = "SELECT COUNT("
           + COL_TRIGGER_NAME + ") FROM "
           + TABLE_PREFIX_SUBST + TABLE_TRIGGERS + " WHERE "
           + COL_SCHEDULER_NAME + " = " + SCHED_NAME_SUBST + " AND NOT ("
           + COL_MISFIRE_INSTRUCTION + " = " + Trigger.MISFIRE_INSTRUCTION_IGNORE_MISFIRE_POLICY + ") AND " 
           + COL_NEXT_FIRE_TIME + " < ? " 
           + "AND " + COL_TRIGGER_STATE + " = ?"; 
```

# 三、DCron实现原理

GitHub: https://git.xiaojukeji.com/gulfstream/dcron.git

采用开源定时任务框架 https://github.com/robfig/cron， 在此基础上进行了封装。

## 3.1 模式架构

**领域模型**

![img](https://cooper.didichuxing.com/uploader/f/3pYjbVbDeqiZAa7T.png?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3MzEwNzc5ODMsImZpbGVHVUlEIjoieGQwQmpJdFFSWWtvWnFwdiIsImlhdCI6MTczMTA3NzM4MywiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwidXNlcklkIjoxMDAwMDAyODY0MH0.fJyBLlFIktnCpHrJvdDpmQ8u4UraIQ4QoHyJvmSJrwg)

**逻辑架构**

dcron分布式服务采用leader和follower的集群策略，leader节点负责调度执行派发任务，follower节点用于同步任务状态和失效备援，这里阐述下leader派发给agent执行任务的逻辑

![img](https://cooper.didichuxing.com/uploader/f/SfELZjY9jo3yHxaC.png?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3MzEwNzc5ODMsImZpbGVHVUlEIjoieGQwQmpJdFFSWWtvWnFwdiIsImlhdCI6MTczMTA3NzM4MywiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwidXNlcklkIjoxMDAwMDAyODY0MH0.fJyBLlFIktnCpHrJvdDpmQ8u4UraIQ4QoHyJvmSJrwg)



**协议设计**

agent模式采用的是标准的grpc通信协议，业务方执行的脚本定时任务都是通过dcron的服务节点grpc调用agent节点来定时执行

agent接收到调用的请求后会非阻塞的方式返回给dcron的服务端，然后执行脚本的定时任务

![img](https://cooper.didichuxing.com/uploader/f/ycXxbJLzejaAlVsX.png?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3MzEwNzc5ODMsImZpbGVHVUlEIjoieGQwQmpJdFFSWWtvWnFwdiIsImlhdCI6MTczMTA3NzM4MywiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwidXNlcklkIjoxMDAwMDAyODY0MH0.fJyBLlFIktnCpHrJvdDpmQ8u4UraIQ4QoHyJvmSJrwg)



## 3.2 调度

调度的主要逻辑：https://git.xiaojukeji.com/gulfstream/dcron/blob/master/dcron/sharding.go

主要利用zk的选主逻辑，同时也会进行主节点计算节点编号、感知节点变更、通知从节点拉取任务等



## 3.2 执行

利用robfig/cron进行定时执行，任务会持久化在内存中，任务的基础信息也会放到localcache中。任务类型上也有shell任务、agent和http任务，执行模式也有单机和广播两种，不支持Map和MapReduce执行。



![img](https://cooper.didichuxing.com/uploader/f/6YPzNtmIz2qNMkm4.png?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3MzEwNzc5ODMsImZpbGVHVUlEIjoieGQwQmpJdFFSWWtvWnFwdiIsImlhdCI6MTczMTA3NzM4MywiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwidXNlcklkIjoxMDAwMDAyODY0MH0.fJyBLlFIktnCpHrJvdDpmQ8u4UraIQ4QoHyJvmSJrwg)

## 3.3 主备

在上线或者机器挂掉的情况下，如果刚好碰见这台机器任务要运行的情况，那这一批任务都会漏执行。

为了减少这种情况，采用了两两互为主备的形式，两个节点会互相backup，通过抢锁保障只有一个机器会执行。当有一个机器在上线或挂掉时，任务就都被转移到另外一个机器上了。

![img](https://cooper.didichuxing.com/uploader/f/gwse19LVFJ38S48W.png?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3MzEwNzc5ODMsImZpbGVHVUlEIjoieGQwQmpJdFFSWWtvWnFwdiIsImlhdCI6MTczMTA3NzM4MywiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwidXNlcklkIjoxMDAwMDAyODY0MH0.fJyBLlFIktnCpHrJvdDpmQ8u4UraIQ4QoHyJvmSJrwg)



# 参考文档

[1] https://blog.csdn.net/Peerless__/article/details/120638890

[2] https://blog.csdn.net/Peerless__/article/details/120675515

[3] https://javaguide.cn/system-design/schedule-task.html#spring-task

[4] https://juejin.cn/post/7056704703559630856

[5] https://www.cnblogs.com/evan-liang/p/14939182.html

[6] http://wiki.intra.xiaojukeji.com/pages/viewpage.action?pageId=841974384