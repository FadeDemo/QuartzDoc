# tutorials-lesson10

quartz在运行前需要配置以下几个模块的东西：

* 线程池

线程池提供线程给quartz执行任务，线程池中线程越多，就能并发地执行越多的任务（理论上说，但有可能你电脑会撑不住:D）。quartz线程池的正确且合适的大小取决于你所使用的 `Scheduler` 的类型。但为了不过多占用资源，线程池的数量越小越好。（确保你的需求的前提下）注意如果线程池的线程数不够， `Trigger` 就会被暂停，这有可能导致线程失火——在配置的 `misfire threshold` 中没有可用的线程。

如果你想要自定义线程池，可以继承 `org.quartz.spi.ThreadPool` 接口。Quartz提供了简单的 `org.quartz.simpl.SimpleThreadPool` 用于使用，它里面的线程数是固定的。

* `JobStore`
* 数据源（如果使用 `JDBCJobStore` ）
* `Scheduler` 本身
  
`Scheduler` 本身需要被设置一个名字、需要 `RMI` 设置、 `JobStore` 的实例和线程池的实例。 `RMI` 设置包括调度程序是否应将自身创建为RMI的服务器对象（使其可用于远程连接），要使用的主机和端口等。

下面是一个远程连接的官方例子：

服务器配置文件 `server.properties` :

```properties
#============================================================================
# Configure Main Scheduler Properties  
#============================================================================

org.quartz.scheduler.instanceName: Sched1
org.quartz.scheduler.rmi.export: true
org.quartz.scheduler.rmi.registryHost: localhost
org.quartz.scheduler.rmi.registryPort: 1099
org.quartz.scheduler.rmi.createRegistry: true

#============================================================================
# Configure ThreadPool  
#============================================================================

org.quartz.threadPool.class: org.quartz.simpl.SimpleThreadPool
org.quartz.threadPool.threadCount: 10
org.quartz.threadPool.threadPriority: 5

#============================================================================
# Configure JobStore  
#============================================================================

org.quartz.jobStore.misfireThreshold: 60000

org.quartz.jobStore.class: org.quartz.simpl.RAMJobStore
```

客户端配置文件 `client.properties` :

```properties
# Properties file for use by StdSchedulerFactory
# to create a Quartz Scheduler Instance.
#

# Configure Main Scheduler Properties  ======================================

org.quartz.scheduler.instanceName: Sched1
org.quartz.scheduler.logger: schedLogger
org.quartz.scheduler.rmi.proxy: true
org.quartz.scheduler.rmi.registryHost: localhost
org.quartz.scheduler.rmi.registryPort: 1099
```

服务端Java代码：

```java
package org.quartz.examples.example12;

import org.quartz.Scheduler;
import org.quartz.SchedulerFactory;
import org.quartz.SchedulerMetaData;
import org.quartz.impl.StdSchedulerFactory;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class RemoteServerExample {

  public void run() throws Exception {
    System.setProperty(StdSchedulerFactory.PROPERTIES_FILE, "org/quartz/examples/example12/server.properties");
    Logger log = LoggerFactory.getLogger(RemoteServerExample.class);

    // First we must get a reference to a scheduler
    SchedulerFactory sf = new StdSchedulerFactory();
    Scheduler sched = sf.getScheduler();

    log.info("------- Initialization Complete -----------");

    log.info("------- (Not Scheduling any Jobs - relying on a remote client to schedule jobs --");

    log.info("------- Starting Scheduler ----------------");

    // start the schedule
    sched.start();

    log.info("------- Started Scheduler -----------------");

    log.info("------- Waiting ten minutes... ------------");

    // wait five minutes to give our jobs a chance to run
    try {
      Thread.sleep(600L * 1000L);
    } catch (Exception e) {
      //
    }

    // shut down the scheduler
    log.info("------- Shutting Down ---------------------");
    sched.shutdown(true);
    log.info("------- Shutdown Complete -----------------");

    SchedulerMetaData metaData = sched.getMetaData();
    log.info("Executed " + metaData.getNumberOfJobsExecuted() + " jobs.");
  }

  public static void main(String[] args) throws Exception {

    RemoteServerExample example = new RemoteServerExample();
    example.run();
  }

}
```

客户端Java代码：

```java
package org.quartz.examples.example12;

import static org.quartz.CronScheduleBuilder.cronSchedule;
import static org.quartz.JobBuilder.newJob;
import static org.quartz.TriggerBuilder.newTrigger;

import org.quartz.JobDataMap;
import org.quartz.JobDetail;
import org.quartz.Scheduler;
import org.quartz.SchedulerFactory;
import org.quartz.Trigger;
import org.quartz.impl.StdSchedulerFactory;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class RemoteClientExample {

    public void run() throws Exception {
        System.setProperty(StdSchedulerFactory.PROPERTIES_FILE, "org/quartz/examples/example12/client.properties");
        Logger log = LoggerFactory.getLogger(RemoteClientExample.class);

        // First we must get a reference to a scheduler
        SchedulerFactory sf = new StdSchedulerFactory();
        Scheduler sched = sf.getScheduler();

        // define the job and ask it to run
        JobDetail job = newJob(SimpleJob.class)
            .withIdentity("remotelyAddedJob", "default")
            .build();
        
        JobDataMap map = job.getJobDataMap();
        map.put("msg", "Your remotely added job has executed!");
        
        Trigger trigger = newTrigger()
            .withIdentity("remotelyAddedTrigger", "default")
            .forJob(job.getKey())
            .withSchedule(cronSchedule("/5 * * ? * *"))
            .build();

        // schedule the job
        sched.scheduleJob(job, trigger);

        log.info("Remote job scheduled.");
    }

    public static void main(String[] args) throws Exception {

        RemoteClientExample example = new RemoteClientExample();
        example.run();
    }

}
```

`Job` Java代码：

```java
package org.quartz.examples.example12;

import java.util.Date;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.quartz.Job;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.quartz.JobKey;

public class SimpleJob implements Job {

    public static final String MESSAGE = "msg";

    private static final Logger logger = LoggerFactory.getLogger(SimpleJob.class);

    public SimpleJob() {
    }

    @Override
    public void execute(JobExecutionContext context)
        throws JobExecutionException {

        // This job simply prints out its job name and the
        // date and time that it is running
        JobKey jobKey = context.getJobDetail().getKey();

        String message = (String) context.getJobDetail().getJobDataMap().get(MESSAGE);

        logger.info("SimpleJob: " + jobKey + " executing at " + new Date());
        logger.info("SimpleJob: msg: " + message);
    }

    

}
```

注意上面的Java代码，要先启动服务器程序再启动客户端程序，不然会报错。其次，服务端程序和客户端程序也都要指明配置文件位置（如 `System.setProperty(StdSchedulerFactory.PROPERTIES_FILE, "org/quartz/examples/example12/client.properties");` ）。最后服务端程序和客户端程序的调度器要使用同一个。

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

