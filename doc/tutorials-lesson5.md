# tutorials-lesson5

`SimpleTrigger` 可以满足的调度需求是：在具体的时间点执行一次，或者在具体的时间点执行，并且以指定的间隔重复执行若干次。

![Snipaste_2021-09-14_14-27-04.png](../img/Snipaste_2021-09-14_14-27-04.png)

`SimpleTrigger` 的属性如上图所示。

`repeatCount` 可以取值为0、整数和 `SimpleTrigger.REPEAT_INDEFINITELY` 。

`repeatInterval` 必须取值为0(官方文档上可以为0，但是实际上貌似不能设置为0)或正的 `long` 类型值，并以毫秒为单位。

`DateBuilder` 对创建 `startTime` 和 `endTime` 来说非常方便


如果设置了 `endTime` 属性，那么它将会覆盖 `repeatCount` 属性。

```java
package org.fade.demo.quartzdemo.tutorialslesson5;

import org.quartz.*;
import org.quartz.impl.StdSchedulerFactory;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.Date;

import static org.quartz.JobBuilder.newJob;
import static org.quartz.SimpleScheduleBuilder.simpleSchedule;

/**
 * @author fade
 * @date 2021/09/14
 */
public class Main {

    private static final Logger logger = LoggerFactory.getLogger(Main.class);

    public static void main(String[] args) {
        try {
            Scheduler scheduler = scheduler = StdSchedulerFactory.getDefaultScheduler();
            scheduler.start();
            // do something
            JobDetail job = newJob(DumbJob.class)
                    .withIdentity("myJob", "group1")
                    .usingJobData("jobSays", "Hello World!")
                    .usingJobData("myFloatValue", 3.141f)
                    .build();
            // 每40秒重复执行一次
            Date date = new Date();
            logger.info("scheduling will end at few seconds after " + date);
            Trigger trigger = TriggerBuilder.newTrigger()
                    .withIdentity("trigger1", "group1")
                    .startNow()
                    .withSchedule(simpleSchedule()
                                    .withIntervalInSeconds(2)
//                            .withRepeatCount(0)
                                    .repeatForever()
                    )
                    .endAt(DateBuilder.nextGivenSecondDate(date, 10))
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

package org.fade.demo.quartzdemo.tutorialslesson5;

import org.quartz.*;

/**
 * @author fade
 * @date 2021/09/08
 */
public class DumbJob implements Job {

    public DumbJob() {
    }

    @Override
    public void execute(JobExecutionContext context)
            throws JobExecutionException
    {
        JobKey key = context.getJobDetail().getKey();

        JobDataMap dataMap = context.getJobDetail().getJobDataMap();

        String jobSays = dataMap.getString("jobSays");
        float myFloatValue = dataMap.getFloat("myFloatValue");

        System.err.println(Thread.currentThread().getName() + "-------" + "Instance " + key + " of DumbJob says: " + jobSays + ", and val is: " + myFloatValue);
    }

}
```

上面的代码中虽然调用了 `repeatForever()` 方法，但后面还调用了 `endAt` 方法，此时任务调度将在指定的结束时间结束。

`SimpleTrigger` 使用 `TriggerBuilder` 设置 `Trigger` 主要的属性，使用 `SimpleScheduleBuilder` 设置与 `SimpleTrigger` 有关的属性。如：

```java
Trigger trigger = TriggerBuilder.newTrigger()
                    .withIdentity("trigger1", "group1")
                    .startNow()
                    .withSchedule(simpleSchedule()
                                    .withIntervalInSeconds(2)
//                            .withRepeatCount(0)
                                    .repeatForever()
                    )
                    .endAt(DateBuilder.nextGivenSecondDate(date, 10))
                    .build();
```

quartz会为你没显式设置的属性设置合理的值。

`SimpleTrigger` 的 `misfireInstruction` 如下所示：

![Snipaste_2021-09-14_16-03-28.png](../img/Snipaste_2021-09-14_16-03-28.png)

![Snipaste_2021-09-14_16-04-23.png](../img/Snipaste_2021-09-14_16-04-23.png)

![Snipaste_2021-09-14_16-05-30.png](../img/Snipaste_2021-09-14_16-05-30.png)

![Snipaste_2021-09-14_16-10-55.png](../img/Snipaste_2021-09-14_16-10-55.png)

在 `SimpleTrigger` 中， `MISFIRE_INSTRUCTION_SMART_POLICY` 会根据配置选择 `MISFIRE_INSTRUCTION_FIRE_NOW` 、 `MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_REMAINING_COUNT` 和 `MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_EXISTING_REPEAT_COUNT` 中的一个，这在 `SimpleTriggerImpl` 的 `updateAfterMisfire` 方法中有所体现：

![Snipaste_2021-09-14_16-18-23.png](../img/Snipaste_2021-09-14_16-18-23.png)

一般我们通过 `SimpleSchedulerBuilder` 的有关方法指定 `Trigger` 对应的 `misfireInstruction` ，如：

![Snipaste_2021-09-14_16-22-12.png](../img/Snipaste_2021-09-14_16-22-12.png)
