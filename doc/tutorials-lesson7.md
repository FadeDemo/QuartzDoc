# tutorials-lesson7

`TriggerListener` 监听与触发器相关的事件， `JobListener` 监听与 `Job` 相关的事件。

与 `Trigger` 有关的事件如下所示：

![Snipaste_2021-09-14_18-34-30.png](../img/Snipaste_2021-09-14_18-34-30.png)

与 `Job` 有关的事件如下所示：

![Snipaste_2021-09-14_18-38-49.png](../img/Snipaste_2021-09-14_18-38-49.png)

创建自定义的监听器只需要实现上面两个接口之一即可，并注册到 `Scheduler` 中，但必须给它们定义一个名字，即给它们的 `getName()` 方法一个返回值

监听器是通过 `Scheduler` 的 `ListenerManager` 注册的

下面是一个示例：

```java
package org.fae.demo.quartzdemo.tutorialslesson7;

import org.quartz.*;
import org.quartz.impl.StdSchedulerFactory;
import org.quartz.impl.matchers.KeyMatcher;

import static org.quartz.JobBuilder.newJob;

/**
 * @author fade
 * @date 2021/09/14
 */
public class Main {

    public static void main(String[] args) {
        try {
            Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
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
            MyJobListener myJobListener = new MyJobListener();
            MyTriggerListener myTriggerListener = new MyTriggerListener();
            scheduler.getListenerManager().addJobListener(myJobListener, KeyMatcher.keyEquals(new JobKey("myJob", "group1")));
//            scheduler.getListenerManager().addJobListenerMatcher("MyJobListener", KeyMatcher.keyEquals(new JobKey("myJob", "group1")));
            scheduler.getListenerManager().addTriggerListener(myTriggerListener, KeyMatcher.keyEquals(new TriggerKey("trigger1", "group1")));
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

package org.fae.demo.quartzdemo.tutorialslesson7;

import org.quartz.JobDetail;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.quartz.JobListener;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * @author fade
 * @date 2021/09/14
 */
public class MyJobListener implements JobListener {

    private static final Logger logger = LoggerFactory.getLogger(MyJobListener.class);

    @Override
    public String getName() {
        return "MyJobListener";
    }

    @Override
    public void jobToBeExecuted(JobExecutionContext context) {
        logger.info("JobDetail " + context.getJobDetail().getKey() + " is about to be executed");
    }

    @Override
    public void jobExecutionVetoed(JobExecutionContext context) {
        logger.info("JobDetail "
                + context.getJobDetail().getKey()
                + " is about to be executed, but a TriggerListener vetoed it's execution.");
    }

    @Override
    public void jobWasExecuted(JobExecutionContext context, JobExecutionException jobException) {
        logger.info("JobDetail " + context.getJobDetail().getKey() + " has been executed");
    }

}

package org.fae.demo.quartzdemo.tutorialslesson7;

import org.quartz.JobDetail;
import org.quartz.JobExecutionContext;
import org.quartz.Trigger;
import org.quartz.TriggerListener;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * @author fade
 * @date 2021/09/14
 */
public class MyTriggerListener implements TriggerListener {

    private static final Logger logger = LoggerFactory.getLogger(MyTriggerListener.class);

    @Override
    public String getName() {
        return "MyTriggerListener";
    }

    @Override
    public void triggerFired(Trigger trigger, JobExecutionContext context) {
        logger.info("Trigger " + trigger.getKey().toString()
                + " has fired, JobDetail " + trigger.getJobKey()
                + " is about to be executed");
    }

    @Override
    public boolean vetoJobExecution(Trigger trigger, JobExecutionContext context) {
//        return true;
        return false;
    }

    @Override
    public void triggerMisfired(Trigger trigger) {
        logger.info("Trigger " + trigger.getKey().toString() + " has misfired");
    }

    @Override
    public void triggerComplete(Trigger trigger, JobExecutionContext context, Trigger.CompletedExecutionInstruction triggerInstructionCode) {
        logger.info("Trigger " + trigger.getKey().toString()
                + " has fired, JobDetail " + trigger.getJobKey()
                + " has been executed");
    }

}
```

需要注意的是 `TriggerListener` 中的 `vetoJobExecution` 如果返回 `true` ，那么 `Job` 中的 `execute()` 方法将不被执行。

既然 `TriggerListener` 和 `JobListener` 可以通过 `Scheduler` 的 `ListenerManager` 来注册，那么也可以从 `ListenerManager` 中移除，移除的方法如下：

![Snipaste_2021-09-15_17-43-18.png](../img/Snipaste_2021-09-15_17-43-18.png)

可以实现在监听器中监听事件从而使一个任务的执行引发另一个任务的执行，下面是quartz官方的一个使用示例：

```java
package org.quartz.examples.example9;

import java.util.Date;

import org.quartz.Job;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.quartz.JobKey;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class SimpleJob1 implements Job {

    private static final Logger logger = LoggerFactory.getLogger(SimpleJob1.class);

    public SimpleJob1() {
    }

    @Override
    public void execute(JobExecutionContext context)
        throws JobExecutionException {

        // This job simply prints out its job name and the
        // date and time that it is running
        JobKey jobKey = context.getJobDetail().getKey();
        logger.info("SimpleJob1 says: " + jobKey + " executing at " + new Date());
    }

}
 
package org.quartz.examples.example9;

import java.util.Date;

import org.quartz.Job;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.quartz.JobKey;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class SimpleJob2 implements Job {

    private static final Logger logger = LoggerFactory.getLogger(SimpleJob2.class);

    public SimpleJob2() {
    }

    @Override
    public void execute(JobExecutionContext context)
        throws JobExecutionException {

        // This job simply prints out its job name and the
        // date and time that it is running
        JobKey jobKey = context.getJobDetail().getKey();
        logger.info("SimpleJob2 says: " + jobKey + " executing at " + new Date());
    }

}
 
package org.quartz.examples.example9;

import static org.quartz.JobBuilder.newJob;
import static org.quartz.TriggerBuilder.newTrigger;

import org.quartz.JobDetail;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.quartz.JobListener;
import org.quartz.SchedulerException;
import org.quartz.Trigger;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * @author wkratzer
 */
public class Job1Listener implements JobListener {

  private static final Logger logger = LoggerFactory.getLogger(Job1Listener.class);

  @Override
  public String getName() {
    return "job1_to_job2";
  }

  @Override
  public void jobToBeExecuted(JobExecutionContext inContext) {
    logger.info("Job1Listener says: Job Is about to be executed.");
  }

  @Override
  public void jobExecutionVetoed(JobExecutionContext inContext) {
    logger.info("Job1Listener says: Job Execution was vetoed.");
  }

  @Override
  public void jobWasExecuted(JobExecutionContext inContext, JobExecutionException inException) {
    logger.info("Job1Listener says: Job was executed.");

    // Simple job #2
    JobDetail job2 = newJob(SimpleJob2.class).withIdentity("job2").build();

    Trigger trigger = newTrigger().withIdentity("job2Trigger").startNow().build();

    try {
      // schedule the job to run!
      inContext.getScheduler().scheduleJob(job2, trigger);
    } catch (SchedulerException e) {
      logger.warn("Unable to schedule job2!");
      e.printStackTrace();
    }

  }

}
 
package org.quartz.examples.example9;

import static org.quartz.JobBuilder.newJob;
import static org.quartz.TriggerBuilder.newTrigger;

import org.quartz.JobDetail;
import org.quartz.JobKey;
import org.quartz.JobListener;
import org.quartz.Matcher;
import org.quartz.Scheduler;
import org.quartz.SchedulerFactory;
import org.quartz.SchedulerMetaData;
import org.quartz.Trigger;
import org.quartz.impl.StdSchedulerFactory;
import org.quartz.impl.matchers.KeyMatcher;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class ListenerExample {

  public void run() throws Exception {
    Logger log = LoggerFactory.getLogger(ListenerExample.class);

    log.info("------- Initializing ----------------------");

    // First we must get a reference to a scheduler
    SchedulerFactory sf = new StdSchedulerFactory();
    Scheduler sched = sf.getScheduler();

    log.info("------- Initialization Complete -----------");

    log.info("------- Scheduling Jobs -------------------");

    // schedule a job to run immediately

    JobDetail job = newJob(SimpleJob1.class).withIdentity("job1").build();

    Trigger trigger = newTrigger().withIdentity("trigger1").startNow().build();

    // Set up the listener
    JobListener listener = new Job1Listener();
    Matcher<JobKey> matcher = KeyMatcher.keyEquals(job.getKey());
    sched.getListenerManager().addJobListener(listener, matcher);

    // schedule the job to run
    sched.scheduleJob(job, trigger);

    // All of the jobs have been added to the scheduler, but none of the jobs
    // will run until the scheduler has been started
    log.info("------- Starting Scheduler ----------------");
    sched.start();

    // wait 30 seconds:
    // note: nothing will run
    log.info("------- Waiting 30 seconds... --------------");
    try {
      // wait 30 seconds to show jobs
      Thread.sleep(30L * 1000L);
      // executing...
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

    ListenerExample example = new ListenerExample();
    example.run();
  }

}
```


