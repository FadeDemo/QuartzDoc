# tutorials-lesson10

quartz在运行前需要配置以下几个模块的东西：

* 线程池

线程池提供线程给quartz执行任务，线程池中线程越多，就能并发地执行越多的任务（理论上说，但有可能你电脑会撑不住:D）。quartz线程池的正确且合适的大小取决于你所使用的 `Scheduler` 的类型。但为了不过多占用资源，线程池的数量越小越好。（确保你的需求的前提下）注意如果线程池的线程数不够， `Trigger` 就会被暂停，这有可能导致线程失火——在配置的 `misfire threshold` 中没有可用的线程。

如果你想要自定义线程池，可以继承 `org.quartz.spi.ThreadPool` 接口。Quartz提供了简单的 `org.quartz.simpl.SimpleThreadPool` 用于使用，它里面的线程数是固定的。

* `JobStore`
* 数据源（如果使用 `JDBCJobStore` ）
* `Scheduler` 本身
  
`Scheduler` 本身需要被设置一个名字、需要 `RMI` 设置、 `JobStore` 的实例和线程池的实例。 `RMI` 设置包括调度程序是否应将自身创建为RMI的服务器对象（使其可用于远程连接），要使用的主机和端口等。

### StdSchedulerFactory

`StdSchedulerFactory` 实现了 `org.quartz.SchedulerFactory` 接口，它使用 `java.util.Properties` 创建和实例化 `Scheduler` 。这些属性既可以保存在文件中，也可以通过程序创建传递给 `SchedulerFactory` 。

一般来说，简单地通过 `StdSchedulerFactory.getDefaultScheduler()` 就可以创建出一个 `Scheduler`

### DirectSchedulerFactory

实现了 `org.quartz.SchedulerFactory` 接口，`DirectSchedulerFactory` 对用代码的方式创建 `Scheduler` 实例非常有用，但是它不允许声明式配置。（配置文件中配置了也没什么用）下面是一个使用 `DirectSchedulerFactory` 的示例：

```java
package org.fade.demo.quartzdemo.tutorialslesson10;

import org.quartz.*;
import org.quartz.impl.DirectSchedulerFactory;

import static org.quartz.JobBuilder.newJob;

/**
 * @author fade
 * @date 2021/09/16
 */
public class Main {

    public static void main(String[] args) {
        try {
            DirectSchedulerFactory factory = DirectSchedulerFactory.getInstance();
            factory.createVolatileScheduler(10);
            Scheduler scheduler = factory.getScheduler();
            scheduler.start();
            // do something
            JobDetail job = newJob(DumbJob.class)
                    .withIdentity("myJob", "group1")
                    .usingJobData("jobSays", "Hello World!")
                    .usingJobData("myFloatValue", 3.141f)
                    .build();
            // 每40秒重复执行一次
            Trigger trigger = TriggerBuilder.newTrigger()
                    .withIdentity("trigger1", "group1")
                    .startNow()
                    .withSchedule(CronScheduleBuilder.cronSchedule("0/2 * * * * ?"))
                    .build();
            scheduler.scheduleJob(job, trigger);
            try {
                Thread.sleep(60000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            scheduler.shutdown();
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
    }

}
```


## 日志

quartz使用slf4j日志框架，如果需要改变日志配置，需要你对slf4j日志框架特别熟悉。如果你想要捕获额外的有关于 `Trigger` 触发和任务执行的信息，你可以考虑启用 `org.quartz.plugins.history.LoggingJobHistoryPlugin` 和/或者 `org.quartz.plugins.history.LoggingTriggerHistoryPlugin`

// TODO: 补充使用示例

