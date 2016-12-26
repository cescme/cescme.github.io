---
title: dubbo应用配置logback日志
date: 2016-12-19 16:15:44
categories: 框架
tags: [DUBBO,日志]
---

logback日志框架可能是目前互联网公司中应用最多的，其百倍于log4j的写入性能保证了其受宠程度。logback基于slf4j的底层进行开发。本文主要结合源码分析下在dubbo应用中怎样正确配置logback进行日志的打印管理

### logback的配置

maven依赖：
```
    <dependency>
      <groupId>ch.qos.logback</groupId>
      <artifactId>logback-classic</artifactId>
      <version>1.1.2</version>
    </dependency>
```

该依赖包会引入logback-classic、logback-core、slf4j-api三个jar包

配置一个最简易的logback.xml
```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <configuration scan="true">
        <property name="APP" value="learn"/>
        <property name="LOG_HOME" value="/export/log/${APP}"/>
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{yy-MM-dd.HH:mm:ss.SSS} [%-16t] %-5p %-22c{0} - %m%n</pattern>
            </encoder>
        </appender>
        
        <logger name="com.alibaba.dubbo" level="DEBUG"/>

        <root level="INFO">
           <appender-ref ref="CONSOLE"/> 
        </root>
    </configuration>
```

将文件放入classpath根路径，如下图所示

![](/images/image011.png)

此时，logback的配置已经生效了，可以正常打印日志。这里需要注意的是logback.xml文件的放置路径一定要是在resource的根目录下，不能在下级目录。logback如果找不到该配置文件，将启用自身的默认配置。

### dubbo中的配置

dubbo的日志管理是通过Log4jLogger来进行的，其核心代码如下
```java
    // 查找常用的日志框架
    static {
        String logger = System.getProperty("dubbo.application.logger");
        if ("slf4j".equals(logger)) {
            setLoggerAdapter(new Slf4jLoggerAdapter());
        } else if ("jcl".equals(logger)) {
            setLoggerAdapter(new JclLoggerAdapter());
        } else if ("log4j".equals(logger)) {
            setLoggerAdapter(new Log4jLoggerAdapter());
        } else if ("jdk".equals(logger)) {
            setLoggerAdapter(new JdkLoggerAdapter());
        } else {
            try {
                setLoggerAdapter(new Log4jLoggerAdapter());
            } catch (Throwable e1) {
                try {
                    setLoggerAdapter(new Slf4jLoggerAdapter());
                } catch (Throwable e2) {
                    try {
                        setLoggerAdapter(new JclLoggerAdapter());
                    } catch (Throwable e3) {
                        setLoggerAdapter(new JdkLoggerAdapter());
                    }
                }
            }
        }
    }
```

```java
    public static void setLoggerAdapter(LoggerAdapter loggerAdapter) {
        if (loggerAdapter != null) {
            Logger logger = loggerAdapter.getLogger(LoggerFactory.class.getName());
            logger.info("using logger: " + loggerAdapter.getClass().getName());
            LoggerFactory.LOGGER_ADAPTER = loggerAdapter;
            for (Map.Entry<String, FailsafeLogger> entry : LOGGERS.entrySet()) {
                entry.getValue().setLogger(LOGGER_ADAPTER.getLogger(entry.getKey()));
            }
        }
    }
```

dubbo会首先检查系统属性dubbo.application.logger，以此确定使用哪种日志框架。该参数我们可以在jvm启动参数中配置，如：

```
    -Ddubbo.application.logger=slf4j
```

这样dubbo在启动后会使用logback进行日志打印，并不关心jar包依赖关系。启动dubbo后，打印的第一条日志如下:

```
    16-12-19.18:25:59.845 [main            ] INFO  LoggerFactory          - using logger: com.alibaba.dubbo.common.logger.slf4j.Slf4jLoggerAdapter
```

这种方式简单明了，但在实际部署中运维给的jvm启动参数往往是固定的，追加参数的话一般流程比较麻烦，所以我们一般还是需要使用下文的方法。

据源码分析，当dubbo.application.logger属性未指定时，dubbo会优先使用log4j，其次才是slf4j。而优先级的顺序是通过异常捕获实现的，即在使用log4j过程中如果出现异常才会使用slf4j。这就给我们提供了思路，当dubbo找不到log4j包里的Logger类，自然会报ClassNotFound异常，这时我们才有机会选用slf4j。因此，我们需要整理下maven的依赖，exclude掉所有的log4j的依赖。

悲剧的是，zkclient(zookeeper的客户端)包依赖于log4j，如果我们将其exclude掉，直接就报ClassNotfound，如果不去掉，dubbo还是会继续使用log4j。

这个时候，需要引入log4j-over-slf4j包，同时将log4j的包exclude掉。

```
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>log4j-over-slf4j</artifactId>
        <version>1.7.6</version>
    </dependency>
```

log4j-over-slf4j实质上是提供了从log4j到slf4j的适配转换功能。其本身的类路径与log4j的类路径是一致的，但具体实现是使用了slf4j的类，是典型的适配器模式。这样，在引入这个包后，我们的zkclient不会再报异常，只是日志打印被转换至slf4j打印。(这里有必要区分下代理模式和适配器模式。代理模式的代理者和被代理者实现的是同一个接口，代理者具体再调用被代理者的真实实现，对于调用方而言是透明的；而适配器模式是实现两个不同接口之间的转换，在这里就是实现从log4j到slf4j的接口实现转换)

这时，我们的项目中没有log4j的依赖了，dubbo将选用slf4j进行打印。

启动dubbo，会发现打印的第一条日志如下：

```
    16-12-19.18:25:59.845 [main            ] INFO  LoggerFactory          - using logger: com.alibaba.dubbo.common.logger.log4j.Log4jLoggerAdapter
```

怎么还是在用log4j呢？不用担心，dubbo检测到的log4j实际是log4j-over-slf4j的适配实现，本质上还是在用slf4j。

