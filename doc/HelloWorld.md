# HelloWorld

```java
package org.fade.demo.quartzdemo.helloworld;

import org.quartz.*;
import org.quartz.impl.StdSchedulerFactory;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import static org.quartz.SimpleScheduleBuilder.simpleSchedule;

/**
 * @author fade
 * @date 2021/09/07
 */
public class QuartzTest {

    public static void main(String[] args) {
        try {
            Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
            scheduler.start();
            // do something
            JobDetail job = JobBuilder.newJob(HelloWorld.class)
                    .withIdentity("job1", "group1")
                    .build();
            // 每40秒重复执行一次
            Trigger trigger = TriggerBuilder.newTrigger()
                    .withIdentity("trigger1", "group1")
                    .startNow()
                    .withSchedule(simpleSchedule()
                            .withIntervalInSeconds(40)
                            .repeatForever())
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

    public static class HelloWorld implements Job {

        Logger logger = LoggerFactory.getLogger(HelloWorld.class);

        @Override
        public void execute(JobExecutionContext jobExecutionContext) throws JobExecutionException {
            logger.info("Hello World with Quartz");
        }

    }

}

```

从上可以看出我们自定义的处理逻辑一般是放在 `Scheduler` 的 `start()` 和 `shutdown()` 方法之间，并且我们可以看出Quartz的几个核心概念：

* `Scheduler`
* `Job`
* `JobDetail`
* `Trigger`

我们将在后面接触Quartz的这几个概念

**注意**

不知道上述代码Job(HelloWorld)的定义你有没有注意，这里是使用的一个静态内部类，因为上面的代码在Quartz中是调用**反射**去创建对象的。

具体可以表现为:

```java
public class SimpleJobFactory implements JobFactory {

    private final Logger log = LoggerFactory.getLogger(getClass());
    
    protected Logger getLog() {
        return log;
    }
    
    public Job newJob(TriggerFiredBundle bundle, Scheduler Scheduler) throws SchedulerException {

        JobDetail jobDetail = bundle.getJobDetail();
        Class<? extends Job> jobClass = jobDetail.getJobClass();
        try {
            if(log.isDebugEnabled()) {
                log.debug(
                    "Producing instance of Job '" + jobDetail.getKey() + 
                    "', class=" + jobClass.getName());
            }
            
            return jobClass.newInstance();
        } catch (Exception e) {
            SchedulerException se = new SchedulerException(
                    "Problem instantiating class '"
                            + jobDetail.getJobClass().getName() + "'", e);
            throw se;
        }
    }

}
```

`public` 修饰的类使通过 `Object instance = clz.getConstructor().newInstance();` 反射创建时可以通过，而static修饰的内部类使其不依赖于外部类的存在而存在，所以通过 `Object instance = clz.getConstructor().newInstance()` 创建对象时也不会报错。但如果内部类是只有 `public` 修饰的话，这时创建内部类需要通过 `Object instance = clz.getDeclaredConstructor(this.getClass()).newInstance(this);` 来创建。

详情可以参考下面这个测试类的运行结果：

```java
package org.fade.demo.quartzdemo.helloworld;

import org.junit.jupiter.api.Test;

import java.lang.reflect.InvocationTargetException;

public class InnerClassReflectCreateTest {

    @Test
    public void testAError() throws ClassNotFoundException, NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException {
        Class<?> clz = Class.forName("org.fade.demo.quartzdemo.helloworld.InnerClassReflectCreateTest$A");
        Object instance = clz.getConstructor().newInstance();
        ((A) instance).test();
    }

    @Test
    public void testARight() throws ClassNotFoundException, NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException {
        Class<?> clz = Class.forName("org.fade.demo.quartzdemo.helloworld.InnerClassReflectCreateTest$A");
        Object instance = clz.getDeclaredConstructor(this.getClass()).newInstance(this);
        ((A) instance).test();
    }

    @Test
    public void testB() throws ClassNotFoundException, NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException {
        Class<?> clz = Class.forName("org.fade.demo.quartzdemo.helloworld.InnerClassReflectCreateTest$B");
        Object instance = clz.getConstructor().newInstance();
        ((B) instance).test();
    }

    @Test
    public void testCError() throws ClassNotFoundException, NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException {
        Class<?> clz = Class.forName("org.fade.demo.quartzdemo.helloworld.C");
        Object instance = clz.getConstructor().newInstance();
        ((C) instance).test();
    }

    @Test
    public void testCRight() throws ClassNotFoundException, NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException {
        Class<?> clz = Class.forName("org.fade.demo.quartzdemo.helloworld.C");
        Object instance = clz.getDeclaredConstructor().newInstance();
        ((C) instance).test();
    }

    public class A {
        void test() {
            System.out.println("A-test");
        }
    }

    public static class B {
        void test() {
            System.out.println("B-test");
        }
    }

}

class C {

    void test() {
        System.out.println("C-test");
    }

}

```

上面的测试类中 `testARight` 、 `testB` 、 `testCRight` 是可以正确通过反射创建对象的。

当然如果你单独创建了一个类文件，那么这个问题就一般不用考虑。