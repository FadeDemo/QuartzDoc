# tutorials-lesson11

### quartz高级功能——集群

quartz的集群功能只在 `JobStore` 是 `JDBCJobstore` ( `JobStoreTX` 或者 `JobStoreCMT` ) 或者 `TerracottaJobStore` 时有效。quartz的集群功能有以下特征：负载均衡和故障转移（如果 `JobDetail` 的“请求恢复”标志设置为 `true` (TODO: 待验证) ）

###### `JobStore` 是 `JDBCJobstore` 时的集群

通过如下配置来开启：

```
org.quartz.jobStore.isClustered=true
```

集群中的每个实例应使用相同的配置文件(也可以使用多份配置文件，但是除了一些允许不同的属性外，其它属性都应该相同)，但是以下是例外：

* 线程池大小（可选）
* `org.quartz.scheduler.instanceId` 的值（必选，但可以设置为 `AUTO` 简单完成）

TODO: 尝试给每个实例配置不同的线程池和instanceId
  
除非系统时钟使用某种形式的时间同步服务（守护进程）进行同步，否则不要在不同的机器上运行集群。

**非集群实例不要和集群实例使用同一组数据库表**，不然会出现严重的数据损坏。

每次触发只有一个节点会执行任务。

下面是使用示例：

```
org.quartz.jobStore.class=org.quartz.impl.jdbcjobstore.JobStoreTX
org.quartz.threadPool.threadCount=10
org.quartz.jobStore.tablePrefix=qrtz_
org.quartz.jobStore.dataSource=quartzdemo
org.quartz.dataSource.quartzdemo.driver=com.mysql.cj.jdbc.Driver
org.quartz.dataSource.quartzdemo.URL=jdbc:mysql:///task_system?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC
org.quartz.dataSource.quartzdemo.user=your-own-db-username
org.quartz.dataSource.quartzdemo.password=your-own-db-password
org.quartz.dataSource.quartzdemo.maxConnections=12
org.quartz.jobStore.driverDelegateClass=org.quartz.impl.jdbcjobstore.StdJDBCDelegate
# 集群配置
org.quartz.jobStore.isClustered=true
org.quartz.scheduler.instanceId=AUTO
```

```java
package org.fade.demo.quartzdemo.tutorialslesson11;

import org.quartz.*;
import org.quartz.impl.StdSchedulerFactory;

import static org.quartz.JobBuilder.newJob;

/**
 * @author fade
 * @date 2021/09/17
 */
public class NodeA {

    public static void main(String[] args) {
        try {
            Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
            scheduler.start();
            // do something
            JobDetail job = newJob(DumbJob.class)
                    .withIdentity("clusterJob", "group1")
                    .usingJobData("jobSays", "Hello World!")
                    .usingJobData("myFloatValue", 3.141f)
                    .build();
            // 避免ObjectAlreadyExistsException
            scheduler.deleteJob(job.getKey());
            // 每40秒重复执行一次
            Trigger trigger = TriggerBuilder.newTrigger()
                    .withIdentity("clusterTrigger", "group1")
                    .startNow()
                    .withSchedule(CronScheduleBuilder.cronSchedule("0/2 * * * * ?"))
                    .build();
            scheduler.scheduleJob(job, trigger);
            try {
                Thread.sleep(120000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            scheduler.shutdown();
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
    }

}

package org.fade.demo.quartzdemo.tutorialslesson11;

import org.quartz.Scheduler;
import org.quartz.SchedulerException;
import org.quartz.impl.StdSchedulerFactory;

/**
 * @author fade
 * @date 2021/09/17
 */
public class NodeB {

    public static void main(String[] args) {
        try {
            Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
            scheduler.start();
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
    }

}

package org.fade.demo.quartzdemo.tutorialslesson11;

import org.quartz.Scheduler;
import org.quartz.SchedulerException;
import org.quartz.impl.StdSchedulerFactory;

/**
 * @author fade
 * @date 2021/09/17
 */
public class NodeC {

    public static void main(String[] args) {
        try {
            Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
            scheduler.start();
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
    }

}
```

请注意不要与非集群方式的数据库混用

上面的示例中，启动三个节点后，任务只会在一个节点中执行，当一个节点挂了后，可以在其它节点中执行任务，但也只在一个节点中执行。

###### `JobStore` 是 `TerracottaJobStore` 时的集群

将 `JobStore` 指定为 `TerracottaJobStore` ，所有的 `Scheduler` 都将会被设置为集群模式

TODO: 补充使用示例

### quartz高级功能——JTA Transactions

可以通过设置 `org.quartz.scheduler.wrapJobExecutionInUserTransaction=true` 使任务在 `JTA` 事务中执行。开启了这个配置后，JTA事务将在Job的execute方法被调用之前开始，并且在执行调用之后commit终止。

如果你希望指定每个任务是否在JTA事务中执行，你可以使用 `@ExecuteInJTATransaction` 注解在 `Job` 类上。

`Scheduler` 有关的方法也能在JTA事务中执行，你所需要保证的是在你调用 `Scheduler` 相关方法时，你已经开启了一个事务。你可以直接通过使用 `UserTransaction` 或将使用调度程序的代码放在使用容器管理事务的SessionBean中来执行此操作。

TODO: 补充使用示例



