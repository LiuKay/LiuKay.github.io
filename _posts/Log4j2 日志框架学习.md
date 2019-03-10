---
layout:     post
title:      Log4j2 学习笔记
subtitle:   Log4j2 学习笔记
date:       2019-01-24
author:     by kayfen
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Java
    - Log4j2
    - 日志
---



# Log4j2 学习笔记

#### 1 Log4j2主要类图

![Log4j2主要类图](https://logging.apache.org/log4j/2.x/images/Log4jClasses.jpg)

*（图片来自于 Apache 官方文档）*

#### 2 Logger 的层次结构

Logger 遵循命名层次结构(`Named Hierarchy`)，比如 name="com.kay" 的 Logger 是 name="com.kay.test" 的父级。

```xml
<Loggers>
    <Root level="INFO">
        <Appender-ref ref="SysConsole"/>
        <Appender-ref ref="ErrorRollingFile"/>
    </Root>
    <Logger name="com.kay" level="INFO" additivity="false">
        <AppenderRef ref="test"/>
        <Appender-ref ref="SysConsole"/>
    </Logger>
</Loggers>
```

Log4j2 中一个`LoggerConfig`对象对应配置文件的中一个`<Logger>`配置，Logger 间的层次关系由 LoggerConfig 来维护，所有 Logger 最顶层的父级 Logger 是 `Root`.

#### 3 Appender

```xml
<Appenders>
    <!-- 默认的控制台输出-->
        <Console name="SysConsole" target="SYSTEM_OUT">
            <ThresholdFilter level="INFO" onMatch="ACCEPT" onMismatch="DENY"/>
            <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n"/>
        </Console>
</Appenders>
```

Appender 代表一个 Logger 的输出地址，比如常见的 控制台(`Console`)，文件等，还可以输出到Socket，数据库等等。一个 Logger 可以指定多个 `Appender`，可以同时输出到不同位置。

`Appender Additivity`

```xml
<Logger name="com.kay" level="info" additivity="true">
    <AppenderRef ref="test1"/>
    <Appender-ref ref="SysConsole"/>
</Logger>
```

Appender 具有可叠加性，或者说是传播性，`additivity`指定了这种特性，默认为true。一般子 Logger 在不指定任何 Appender 的情况下会继承父级的 Appender，如果在父级 Logger 中同时指定了 同一个 Appender ，并且没有指定`additivity=false`，则会在同一个地方输出2次相同的日志。

#### 4 Layout 指定输出格式

Layout 有多种形式 (JSON，HTML，CSV 等)，可以参见 官方文档[`Layout`](https://logging.apache.org/log4j/2.x/manual/layouts.html)

常用的是 `Pattern Layout`，举例：

`%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n`会输出如下格式日志：

```verilog
11:40:30.458 [main] INFO  org.apache.catalina.core.StandardEngine - Starting Servlet Engine: Apache Tomcat/8.5.29
```

`每个转换说明符以百分号(%)开头`，后跟可选格式修饰符和转换字符，以下是`部分格式`：

```properties
p (level) 日志级别
c（logger） Logger的Name
C (class) Logger调用者的全限定类名 ***
d (date) 日期
highlight 高亮颜色
l (location) 调用位置 ***
L (line) 行号
m (msg/message) 输出的内容
M (methode) 调用方法 ***
maker marker的全限定名
n 输出平台相关的换行符,如'\n' '\r\n'
pid (processId) 进程ID
level （p）日志级别
r JVM启动后经过的微秒
t (tn/thread/threadName) 线程名称
T (tid/threadId) 线程ID
tp (threadPriority) 线程优先级
x (NDC) 线程Context堆栈
```

> 注：
>
> 1.括号里为同等效果的单词写法。
>
> 2.带`***`结尾的输出格式，官方说`is an expensive operation and may impact performance. Use with caution.`开销比较大，可能会带来性能问题，请谨慎使用。

#### 4 RollingFileAppender 设置日志文件滚动输出

日志文件滚动输出需要设置一个触发方案(`Triggering Policie`)和 滚动策略(`RolloverStrategy`)。

#### 4.1 Triggering Policies

##### TimeBasedTriggeringPolicy 基于时间的触发策略

配置时间周期进行日志的滚动输出，该触发策略有3个属性设置：

- interval : 在指定时间单位的触发间隔，默认值为`1`。比如时间单位是小时，则每间隔1小时滚动一次。
- modulate :  是否在间隔的时间边界触发滚动，默认`false`。比如时间单位是小时，interval=4，如果当前时间是3:00，开启modulate=true，则第一次出发在4:00，第二次是8:00，以此类推。
- maxRandomDelay  : 表示最大延迟时间，默认`0`，表示无延迟。

##### SizeBasedTriggeringPolicy 基于文件大小的触发策略

可配置单位有：byte，KB，MB ，GB。如指定`10MB`

以上为常用触发策略，此外还有混合触发策略(`CompositeTriggeringPolicy`)，Cron表达式的触发(`CronTriggeringPolicy`)，启动触发(`OnStartupTriggeringPolicy`)等，具体可以参考官网。

#### 4.2 RolloverStrategy

##### Delete on Rollover 配置过期删除

```xml
<RollingFile name="testCommon" fileName="${logPath}/app.log"
                     filePattern="${historyLogPath}/$${date:yyyy-MM}/app-%d{yyyy-MM-dd}-%i.log.gz">
            <PatternLayout pattern="%d{yyyy.MM.dd HH:mm:ss z} %-5level %c{36} - %msg%n"/>
            <Policies>
                <TimeBasedTriggeringPolicy />
                <SizeBasedTriggeringPolicy size="1M" />
            </Policies>
            <DefaultRolloverStrategy max="20">
                <Delete basePath="${historyLogPath}" maxDepth="2">
                    <!-- 配置超过15天的日志压缩文件删除-->
                    <IfFileName glob="*/app-*.log.gz">
                        <IfLastModified age="15d" />
                    </IfFileName>
                </Delete>
            </DefaultRolloverStrategy>
        </RollingFile>
```

#### 异步日志

log42-2.9或更高版本需要添加 `disruptor-3.3.4.jar`或更高版本依赖到classpath下.

异步日志器默认不会打印位置信息，如下测试配置文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="info" monitorInterval="5" name="test1">
    <Appenders>
        <Console name="SysConsole1" target="SYSTEM_OUT">
            <ThresholdFilter level="INFO" onMatch="ACCEPT" onMismatch="DENY"/>
            <PatternLayout pattern="root:%d{HH:mm:ss.SSS} [%t] %-5level %class{36} - %msg%n"/>
        </Console>
 
        <Console name="SysConsole2" target="SYSTEM_OUT">
            <ThresholdFilter level="INFO" onMatch="ACCEPT" onMismatch="DENY"/>
            <PatternLayout pattern="asyn:%d{HH:mm:ss.SSS} [%t] %-5level %class{36} - %msg%n"/>
        </Console>
    </Appenders>
    <Loggers>
        <Root level="INFO">
            <Appender-ref ref="SysConsole1"/>
        </Root>
 
        <AsyncLogger name="com.spbt.log" level="INFO">
            <Appender-ref ref="SysConsole2"/>
        </AsyncLogger>
    </Loggers>
</Configuration>
```

(`%class`指定打印日志的类位置)

测试结果打印出如下信息：

![](http://ww1.sinaimg.cn/large/78c3a6d7ly1fzhhvwyzbwj20he02zmxu.jpg)

如上，并没有打印位置相关信息。如需要打印，可在`AsynLogger`添加属性`includeLocation="true"`，但是会损失一定的性能。log42的异步日志主打高吞吐量和高性能，打印位置信息会查询堆栈信息会带来额外的开销。

##### 最后附上一般日志配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="info" monitorInterval="5" name="test1">
    <properties>
        <!--设置日志在硬盘上输出的目录-->
        <property name="logPath">log</property>
        <property name="historyLogPath">logs</property>
    </properties>
    <Appenders>
        <Console name="SysConsole" target="SYSTEM_OUT">
            <ThresholdFilter level="INFO" onMatch="ACCEPT" onMismatch="DENY"/>
            <PatternLayout pattern="%d{HH:mm:ss.SSS} %-5level [%t]  %c{36} - %msg%n"/>
        </Console>
 
        <RollingFile name="RollingFile" fileName="${logPath}/app.log"
                     filePattern="${historyLogPath}/$${date:yyyy-MM}/app-%d{MM-dd-yyyy}-%i.log.gz">
            <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss z} %-5level [%t] %c{36} - %msg%n"/>
            <Policies>
                <TimeBasedTriggeringPolicy modulate="true"  interval="1" />
                <SizeBasedTriggeringPolicy size="10M" />
            </Policies>
            <DefaultRolloverStrategy>
                <!-- 日志删除策略,过期30天删除-->
                <Delete basePath="${historyLogPath}" maxDepth="2">
                    <IfFileName glob="*/app-*.log.gz">
                        <IfLastModified age="30d" />
                    </IfFileName>
                </Delete>
            </DefaultRolloverStrategy>
        </RollingFile>
 
        <RollingFile name="ErrorRollingFile" filename="${logPath}/error.log"
                     filepattern="${historyLogPath}/%d{YYYYMMdd}-%i-error.log.zip">
            <!--设置只输出级别为ERROR的日志-->
            <ThresholdFilter level="ERROR"/>
            <PatternLayout pattern="%d{YYYY-MM-dd HH:mm:ss.SSS} %-5level [%t] %l - %msg%n" />
            <Policies>
                <TimeBasedTriggeringPolicy interval="1" modulate="true"/>
                <SizeBasedTriggeringPolicy size="10M" />
            </Policies>
            <DefaultRolloverStrategy />
        </RollingFile>
 
    </Appenders>
 
    <Loggers>
        <Root level="INFO">
            <Appender-ref ref="SysConsole"/>
        </Root>
 
        <!--异步日志-->
        <!--<AsyncLogger name="com.spbt.log" level="INFO">-->
            <!--<Appender-ref ref="SysConsole"/>-->
        <!--</AsyncLogger>-->
    </Loggers>
</Configuration>
```

