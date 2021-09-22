# tutorials-lesson12

quartz的其它功能有：

* 插件

如果想要自定义插件，可以实现 `org.quartz.spi.SchedulerPlugin` 接口。

当然在 `org.quartz.plugins` 包下也有quartz自定义的插件：

![Snipaste_2021-09-17_14-57-54.png](../img/Snipaste_2021-09-17_14-57-54.png)

下面是quartz官方的一个使用示例：

配置文件 `quartz.properties`

```properties

#============================================================================
# Configure Main Scheduler Properties  
#============================================================================

org.quartz.scheduler.instanceName: TestScheduler
org.quartz.scheduler.instanceId: AUTO

#============================================================================
# Configure ThreadPool  
#============================================================================

org.quartz.threadPool.class: org.quartz.simpl.SimpleThreadPool
org.quartz.threadPool.threadCount: 3
org.quartz.threadPool.threadPriority: 5

#============================================================================
# Configure JobStore  
#============================================================================

org.quartz.jobStore.misfireThreshold: 60000

org.quartz.jobStore.class: org.quartz.simpl.RAMJobStore

#============================================================================
# Configure Plugins 
#============================================================================

org.quartz.plugin.triggHistory.class: org.quartz.plugins.history.LoggingJobHistoryPlugin

org.quartz.plugin.jobInitializer.class: org.quartz.plugins.xml.XMLSchedulingDataProcessorPlugin
org.quartz.plugin.jobInitializer.fileNames: org/quartz/examples/example10/quartz_data.xml
org.quartz.plugin.jobInitializer.failOnFileNotFound: true
org.quartz.plugin.jobInitializer.scanInterval: 120
org.quartz.plugin.jobInitializer.wrapInUserTransaction: false
```

配置文件 `quartz_data.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<job-scheduling-data xmlns="http://www.quartz-scheduler.org/xml/JobSchedulingData"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.quartz-scheduler.org/xml/JobSchedulingData http://www.quartz-scheduler.org/xml/job_scheduling_data_1_8.xsd"
    version="1.8">
    
    <pre-processing-commands>
        <delete-jobs-in-group>*</delete-jobs-in-group>  <!-- clear all jobs in scheduler -->
        <delete-triggers-in-group>*</delete-triggers-in-group> <!-- clear all triggers in scheduler -->
    </pre-processing-commands>
    
    <processing-directives>
        <!-- if there are any jobs/trigger in scheduler of same name (as in this file), overwrite them -->
        <overwrite-existing-data>true</overwrite-existing-data>
        <!-- if there are any jobs/trigger in scheduler of same name (as in this file), and over-write is false, ignore them rather then generating an error -->
        <ignore-duplicates>false</ignore-duplicates> 
    </processing-directives>
    
    <schedule>
	    <job>
	        <name>TestJob1</name>
	        <job-class>org.quartz.examples.example10.SimpleJob</job-class>
	    </job>
	    
        <job>
            <name>TestDurableJob</name>
            <job-class>org.quartz.examples.example10.SimpleJob</job-class>
            <durability>true</durability>
            <recover>false</recover>
        </job>
	    
	    <trigger>
	        <simple>
	            <name>TestSimpleTrigger1AtFiveSecondInterval</name>
	            <job-name>TestJob1</job-name>
	            <repeat-count>-1</repeat-count> <!-- repeat indefinitely  -->
	            <repeat-interval>5000</repeat-interval>  <!--  every 5 seconds -->
	        </simple>
	    </trigger>
	
	    <job>
	        <name>TestJob2</name>
	        <group>GroupOfTestJob2</group>
	        <description>This is the description of TestJob2</description>
	        <job-class>org.quartz.examples.example10.SimpleJob</job-class>
	        <durability>false</durability>
	        <recover>true</recover>
	        <job-data-map>
	            <entry>
	                <key>someKey</key>
	                <value>someValue</value>
	            </entry>
	            <entry>
	                <key>someOtherKey</key>
	                <value>someOtherValue</value>
	            </entry>
	        </job-data-map>
	    </job>
	    
	    <trigger>
	        <simple>
	            <name>TestSimpleTrigger2AtTenSecondIntervalAndFiveRepeats</name>
	            <group>GroupOfTestJob2Triggers</group>
	            <job-name>TestJob2</job-name>
	            <job-group>GroupOfTestJob2</job-group>
	            <start-time>2010-02-09T10:15:00</start-time>
	            <misfire-instruction>MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_EXISTING_REPEAT_COUNT</misfire-instruction>
	            <repeat-count>5</repeat-count>
	            <repeat-interval>10000</repeat-interval>
	        </simple>
	    </trigger>
	    
	    <trigger>
	        <cron>
	            <name>TestCronTrigger2AtEveryMinute</name>
	            <group>GroupOfTestJob2Triggers</group>
	            <job-name>TestJob2</job-name>
	            <job-group>GroupOfTestJob2</job-group>
                <job-data-map>
                    <entry>
                        <key>someKey</key>
                        <value>overriddenValue</value>
                    </entry>
                    <entry>
                        <key>someOtherKey</key>
                        <value>someOtherOverriddenValue</value>
                    </entry>
                </job-data-map>
                <cron-expression>0 * * ? * *</cron-expression>
	        </cron>
	    </trigger>
	
	    <trigger>
	        <cron>
	            <name>TestCronTrigger2AtEveryMinuteOnThe45thSecond</name>
	            <group>GroupOfTestJob2Triggers</group>
	            <job-name>TestJob2</job-name>
	            <job-group>GroupOfTestJob2</job-group>
	            <start-time>2010-02-09T12:26:00.0</start-time>
	            <end-time>2012-02-09T12:26:00.0</end-time>
	            <misfire-instruction>MISFIRE_INSTRUCTION_SMART_POLICY</misfire-instruction>
	            <cron-expression>45 * * ? * *</cron-expression>
	            <time-zone>America/Los_Angeles</time-zone>
	        </cron>
	    </trigger>
    </schedule>    
</job-scheduling-data>
```

Java代码

```java
package org.quartz.examples.example10;

import org.quartz.Scheduler;
import org.quartz.SchedulerFactory;
import org.quartz.SchedulerMetaData;
import org.quartz.impl.StdSchedulerFactory;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * This example will spawn a large number of jobs to run
 * 
 * @author James House, Bill Kratzer
 */
public class PlugInExample {

  public void run() throws Exception {
    // 设置配置文件路径（如果是直接放在类路径顶级目录下无需此操作）
    System.setProperty(StdSchedulerFactory.PROPERTIES_FILE, "org/quartz/examples/example10/quartz.properties");
    Logger log = LoggerFactory.getLogger(PlugInExample.class);

    // First we must get a reference to a scheduler
    SchedulerFactory sf = new StdSchedulerFactory();
    Scheduler sched;
    try {
      sched = sf.getScheduler();
    } catch (NoClassDefFoundError e) {
      log.error(" Unable to load a class - most likely you do not have jta.jar on the classpath. If not present in the examples/lib folder, please " +
                "add it there for this sample to run.", e);
      return;
    }

    log.info("------- Initialization Complete -----------");

    log.info("------- (Not Scheduling any Jobs - relying on XML definitions --");

    log.info("------- Starting Scheduler ----------------");

    // start the schedule
    sched.start();

    log.info("------- Started Scheduler -----------------");

    log.info("------- Waiting five minutes... -----------");

    // wait five minutes to give our jobs a chance to run
    try {
      Thread.sleep(300L * 1000L);
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

    PlugInExample example = new PlugInExample();
    example.run();
  }

}
 
package org.quartz.examples.example10;

import java.util.Date;
import java.util.Set;

import org.quartz.Job;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.quartz.JobKey;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class SimpleJob implements Job {

    private static final Logger logger = LoggerFactory.getLogger(SimpleJob.class);

    public SimpleJob() {
    }

    @Override
    public void execute(JobExecutionContext context)
        throws JobExecutionException {

        // This job simply prints out its job name and the
        // date and time that it is running
        JobKey jobKey = context.getJobDetail().getKey();
        logger.info("Executing job: " + jobKey + " executing at " + new Date() + ", fired by: " + context.getTrigger().getKey());
        
        if(context.getMergedJobDataMap().size() > 0) {
            Set<String> keys = context.getMergedJobDataMap().keySet();
            for(String key: keys) {
                String val = context.getMergedJobDataMap().getString(key);
                logger.info(" - jobDataMap entry: " + key + " = " + val);
            }
        }
        
        context.setResult("hello");
    }

}
```

注意上面的示例有几个关键点：

首先 `System.setProperty(StdSchedulerFactory.PROPERTIES_FILE, "org/quartz/examples/example10/quartz.properties");` 和 `org.quartz.plugin.jobInitializer.fileNames: org/quartz/examples/example10/quartz_data.xml` 这两段代码的作用是使quartz可以读取到配置文件，因为quartz默认是从classpath的根目录下或quartz自带的目录下取配置文件的。其次还需要导入 `JTA 事务` 的 maven 依赖：

```xml
<dependency>
    <groupId>javax.transaction</groupId>
    <artifactId>jta</artifactId>
    <version>jta-latest-version</version>
</dependency>
```

* `JobFactory` 

当任务被触发时，是通过配置在 `Scheduler` 上的 `JobFactory` 实例化 `Job` 的。默认的 `JobFactory` 只是通过反射的 `newInstance()` 方法创建，如果想要实现依赖注入和控制反转功能，你可以通过实现 `JobFactory` 接口自定义 `JobFactory`

`Scheduler` 有一个 `setJobFactory` 方法用来设置 `JobFactory`

* quartz提供的 `Job`

quartz也提供了一系列实用的 `Job` ，这些 `Job` 可以在 `org.quartz.jobs` 包中找到，但是这需要额外导入一个maven依赖：

```
<dependency>
    <groupId>org.quartz-scheduler</groupId>
    <artifactId>quartz-jobs</artifactId>
    <version>${quartz-version}</version>
</dependency>
```

![Snipaste_2021-09-17_15-20-23.png](../img/Snipaste_2021-09-17_15-20-23.png)