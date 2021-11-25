# 集成SpringBoot

本文主要介绍Quartz集成SpringBoot的简单应用

# 依赖

Quartz集成Spring Boot大致需要以下依赖（maven）：

```xml
<dependency>
  <groupId>mysql</groupId>
  <artifactId>mysql-connector-java</artifactId>
  <version>${mysql-version}</version>
</dependency>

<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-quartz</artifactId>
  <version>${spring.boot.version}</version>
</dependency>

<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-jdbc</artifactId>
  <version>${spring.boot.version}</version>
</dependency>
```

### 示例

简单的一个任务，打印 `Hello Spring Boot, I am Quartz!` 三次，中间间隔5秒

* Spring Boot 配置文件

```yml
spring:
  profiles:
    # 定义你自己的application-db.yml
    # 以此来引用如mysql-driver-class-name之类的变量
    active: db
  datasource:
    driver-class-name: ${mysql-driver-class-name}
    url: ${mysql-url}
    username: ${mysql-username}
    password: ${mysql-password}
  quartz:
    job-store-type: jdbc
    properties:
      jobStore:
        class: org.quartz.impl.jdbcjobstore.JobStoreTX
        driverDelegateClass: org.quartz.impl.jdbcjobstore.StdJDBCDelegate
        tablePrefix: QRTZ_
        useProperties: false
      threadPool:
        class: org.quartz.simpl.SimpleThreadPool
        threadCount: 10
        threadPriority: 5
```

* Spring Boot 启动类

```java
package org.fade.demo.quartzdemo.quartzspringboot;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
 * @author fade
 * @date 2021/11/24
 */
@SpringBootApplication
public class QuartzSpringBootApplication {

    public static void main(String[] args) {
        // todo 理解quartz的持久化，
        //  为什么trigger的信息只在misfired时持久化到数据库中
        SpringApplication.run(QuartzSpringBootApplication.class, args);
    }

}
```

* Quartz配置类

```java
package org.fade.demo.quartzdemo.quartzspringboot;

import org.quartz.JobDetail;
import org.quartz.Trigger;
import org.quartz.TriggerBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import static org.quartz.JobBuilder.newJob;
import static org.quartz.SimpleScheduleBuilder.simpleSchedule;

/**
 * @author fade
 * @date 2021/11/25
 */
@Configuration
public class QuartzConfig {

    @Bean
    public JobDetail jobDetail() {
        return newJob(HelloSpringBoot.class)
                .storeDurably()
                .build();
    }

    @Bean
    public Trigger trigger(JobDetail jobDetail) {
        return TriggerBuilder.newTrigger().startNow()
                .withSchedule(simpleSchedule()
                .withIntervalInSeconds(5)
                .withRepeatCount(2))
                .forJob(jobDetail)
                .build();
    }

}
```

* Job

```java
package org.fade.demo.quartzdemo.quartzspringboot;

import org.quartz.Job;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * @author fade
 * @date 2021/11/24
 */
public class HelloSpringBoot implements Job {

    private static final Logger LOG = LoggerFactory.getLogger(HelloSpringBoot.class);

    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        LOG.info("Hello Spring Boot, I am Quartz!");
    }

}
```

这里集成Spring Boot时也和集成Spring时一样，需要给 `Trigger` 关联对应的 `JobDetail`

// TODO: 理解quartz的持久化，为什么trigger的信息只在misfired时持久化到数据库中