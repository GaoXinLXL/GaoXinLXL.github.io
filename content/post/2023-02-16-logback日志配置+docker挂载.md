---
title: logback日志配置+docker挂载

date: '2023-02-16'
categories:
    - 笔记
tags:
    - Java
    - MySQL
---

> 背景：日志配置文件`logback-spring.xml` 、线上`docker` 部署
> 需求：日志能够在linux机器指定路径上查阅
>

# 1.日志文件配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <property name="CONSOLE_LOG_PATTERN" value="%d{yyyy-MM-dd HH:mm:ss.SSS} %5p --- [%15.15t] %-80.80logger{79} [%line] : %m%n"/>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>debug</level>
        </filter>
        <encoder>
            <Pattern>${CONSOLE_LOG_PATTERN}</Pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <appender name="INFO_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>/home/data/app/plg/log/spring.log</file>
        <encoder>
            <pattern>${CONSOLE_LOG_PATTERN}</pattern>
            <charset>UTF-8</charset> <!-- 设置字符集 -->
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>/home/data/app/plg/log/spring.log.%d{yyyy-MM-dd}.%i.gz</fileNamePattern>
            <maxFileSize>800MB</maxFileSize>
            <maxHistory>10</maxHistory>
            <totalSizeCap>30GB</totalSizeCap>
        </rollingPolicy>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <root level="INFO">
        <appender-ref ref="INFO_FILE"/>
        <appender-ref ref="CONSOLE"/>
    </root>

</configuration>
```

重点关注该配置：

```xml
<file>/home/data/app/plg/log/spring.log</file>
```

指明了日志文件在服务器中的路径为`/home/data/app/plg/log/spring.log` 。但要注意，由于项目由docker部署，日志会保存在docker对应项目容器的该路径下，docker容器重启会丢失数据，所以需要需要将这部分信息持久化保存到服务器上。借助docker挂载完成日志文件的持久化。

# 2.docker挂载配置

项目使用docker-compose管理，配置文件中关于挂载部分如下：

```xml
volumes:
      - /home/data/plg/logs/spring_logs:/home/data/app/
```

将容器中/home/data/app/下的数据，挂载到服务器的/home/data/plg/logs/spring_logs路径下

# 3.结果

最终，项目产生的日志会落在容器该位置/home/data/app/plg/log/spring.log，根据挂载配置，会被挂载到服务器的该位置/home/data/plg/logs/spring_logs/plg/log/spring.log

后续不用进入容器，直接查服务器对应位置日志就行。

# 4.总结

- `logback-spring.xml` 中配置的日志地址，位于服务器对应位置，如果是docker部署，就进入docker查看对应位置
- `docker-compose.yml`中配置的挂载，将容器内的数据持久化到服务器对应路径下
- 后续查日志，直接查服务器对应位置下的日志就行
