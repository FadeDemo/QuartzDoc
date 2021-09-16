# tutorials-lesson1

代码和前面[HelloWorld](HelloWorld.md)的类似(来源于官网)。

```java
SchedulerFactory schedFact = new org.quartz.impl.StdSchedulerFactory();

Scheduler sched = schedFact.getScheduler();

sched.start();

// define the job and tie it to our HelloJob class
JobDetail job = newJob(HelloJob.class)
      .withIdentity("myJob", "group1")
      .build();

// Trigger the job to run now, and then every 40 seconds
Trigger trigger = newTrigger()
      .withIdentity("myTrigger", "group1")
      .startNow()
      .withSchedule(simpleSchedule()
          .withIntervalInSeconds(40)
          .repeatForever())
      .build();

// Tell quartz to schedule the job using our trigger
sched.scheduleJob(job, trigger);
```

从官网的这段话再结合上面的代码，我们大概可以知道 `Scheduler` 的概念

> Once a scheduler is instantiated, it can be started, placed in stand-by mode, and shutdown. Note that once a scheduler is shutdown, it cannot be restarted without being re-instantiated. Triggers do not fire (jobs do not execute) until the scheduler has been started, nor while it is in the paused state.

`Scheduler` 是一个调度容器，里面可以注册多个 `JobDetail` 和 `Trigger` 。当 `Trigger` 与 `JobDetail` 组合，就可以被 Scheduler 容器调度了。