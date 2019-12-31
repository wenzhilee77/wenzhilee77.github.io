---
layout: post
title:  Java Logback
categories: Java Logback
tags:  Java Logback
author: wenzhilee77
---

# logack 简介

目前还没有看过日志类框架的源码，仅限于如何使用。所以就不说那些“空话”了。最直观的认知是：

1. logback和log4j是一个人写的
2. springboot默认使用的日志框架是logback。
3. 三个模块组成
   * logback-core
   * logback-classic
   * logback-access
logback-core 是其它模块的基础设施，其它模块基于它构建，显然，logback-core 提供了一些关键的通用机制。logback-classic 的地位和作用等同于 Log4J，它也被认为是 Log4J 的一个改进版，并且它实现了简单日志门面 SLF4J；而 logback-access 主要作为一个与 Servlet 容器交互的模块，比如说tomcat或者 jetty，提供一些与 HTTP 访问相关的功能。

# 配置文件详解

## configuration
先来看这张图，这个结构就是整个logback.xml配置文件的结构。

![](/images/logback/001.jpg)

```xml
<configuration scan="true" scanPeriod="60 seconds" debug="false">  
    <property name="glmapper-name" value="glmapper-demo" /> 
    <contextName>${glmapper-name}</contextName> 
    
    
    <appender>
        //xxxx
    </appender>   
    
    <logger>
        //xxxx
    </logger>
    
    <root>             
       //xxxx
    </root>  
</configuration>  
```

ps：想使用spring扩展profile支持，要以logback-spring.xml命名，其他如property需要改为springProperty

* scan:当此属性设置为true时，配置文件如果发生改变，将会被重新加载，默认值为true。
* scanPeriod:设置监测配置文件是否有修改的时间间隔，如果没有给出时间单位，默认单位是毫秒。当scan为true时，此属性生效。默认的时间间隔为1分钟。
* debug:当此属性设置为true时，将打印出logback内部日志信息，实时查看logback运行状态。默认值为false。

1. contextName

每个logger都关联到logger上下文，默认上下文名称为“default”。但可以使用contextName标签设置成其他名字，用于区分不同应用程序的记录

2. property

用来定义变量值的标签，property标签有两个属性，name和value；其中name的值是变量的名称，value的值时变量定义的值。通过property定义的值会被插入到logger上下文中。定义变量后，可以使“${name}”来使用变量。如上面的xml所示。

3. logger

用来设置某一个包或者具体的某一个类的日志打印级别以及指定appender。

4. root

根logger，也是一种logger，且只有一个level属性

5. appender

负责写日志的组件，下面会细说

6. filter

filter其实是appender里面的子元素。它作为过滤器存在，执行一个过滤器会有返回DENY，NEUTRAL，ACCEPT三个枚举值中的一个。

* DENY：日志将立即被抛弃不再经过其他过滤器
* NEUTRAL：有序列表里的下个过滤器过接着处理日志
* ACCEPT：日志会被立即处理，不再经过剩余过滤器

# 案例分析
首先来配置一个非常简单的文件。这里申请下，我使用的是 logback-spring.xml。和 logback.xml 在properties上有略微差别。其他都一样。

properties中就是指定了日志的打印级别和日志的输出位置：
```java
#设置应用的日志级别
logging.level.com.glmapper.spring.boot=INFO
#路径
logging.path=./logs
```

## 通过控制台输出的log
logback-spring.xml的配置如下：
```xml
<configuration>
    <!-- 默认的控制台日志输出，一般生产环境都是后台启动，这个没太大作用 -->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <Pattern>%d{HH:mm:ss.SSS} %-5level %logger{80} - %msg%n</Pattern>
        </encoder>
    </appender>
    
    <root level="info">
        <appender-ref ref="STDOUT"/>
    </root>
</configuration>
```

打印日志的controller
```java
private static final Logger LOGGER =
LoggerFactory.getLogger(HelloController.class);
@Autowired
private TestLogService testLogService;

@GetMapping("/hello")
public String hello(){
    LOGGER.info("GLMAPPER-SERVICE:info");
    LOGGER.error("GLMAPPER-SERVICE:error");
    testLogService.printLogToSpecialPackage();
    return "hello spring boot";
}
```

验证结果：
```java
01:50:39.633 INFO  com.glmapper.spring.boot.controller.HelloController
- GLMAPPER-SERVICE:info
01:50:39.633 ERROR com.glmapper.spring.boot.controller.HelloController
- GLMAPPER-SERVICE:error
```

## 控制台不打印，直接输出到日志文件
先来看下配置文件：
```xml
<configuration>
    <!-- 属性文件:在properties文件中找到对应的配置项 -->
    <springProperty scope="context" name="logging.path"  source="logging.path"/>
    <springProperty scope="context" name="logging.level" source="logging.level.com.glmapper.spring.boot"/>
    <!-- 默认的控制台日志输出，一般生产环境都是后台启动，这个没太大作用 -->
    <appender name="STDOUT"
        class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <Pattern>%d{HH:mm:ss.SSS} %-5level %logger{80} - %msg%n</Pattern>
        </encoder>
    </appender>
    
    <appender name="GLMAPPER-LOGGERONE"
    class="ch.qos.logback.core.rolling.RollingFileAppender">
        <append>true</append>
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>${logging.level}</level>
        </filter>
        <file>
            ${logging.path}/glmapper-spring-boot/glmapper-loggerone.log
        </file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <FileNamePattern>${logging.path}/glmapper-spring-boot/glmapper-loggerone.log.%d{yyyy-MM-dd}</FileNamePattern>
            <MaxHistory>30</MaxHistory>
        </rollingPolicy>
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>
    
    <root level="info">
        <appender-ref ref="GLMAPPER-LOGGERONE"/>
    </root>
</configuration>
```

这里我们appender-ref指定的appender是GLMAPPER-LOGGERONE，因为之前没有名字为GLMAPPER-LOGGERONE的appender，所以要增加一个name为GLMAPPER-LOGGERONE的appender。
注意上面这个配置，我们是直接接将root的appender-ref直接指定到我们的GLMAPPER-LOGGERONE这个appender的。所以控制台中将只会打印出bannar之后就啥也不打印了，所有的启动信息都会被打印在日志文件glmapper-loggerone.log中。

但是实际上我们不希望我的业务日志中会包括这些启动信息。所以这个时候我们就需要通过logger标签来搞事情了。将上面的配置文件进行简单修改：

```xml
<logger name="com.glmapper.spring.boot.controller" level="${logging.level}"
        additivity="false">
    <appender-ref ref="GLMAPPER-LOGGERONE" />
</logger>

<root level="${logging.level}">
    <appender-ref ref="STDOUT"/>
</root>
```

让root指向控制台输出；logger负责打印包com.glmapper.spring.boot.controller下的日志。

验证结果

还是通过我们的测试controller来打印日志为例，但是这里不会在控制台出现日志信息了。期望的日志文件在./logs/glmapper-spring-boot/glmapper-loggerone.log。
![](/images/logback/002.jpg)

## logger和appender的关系
上面两种是一个基本的配置方式，通过上面两个案例，我们先来了解下logger/appender/root之间的关系，然后再详细的说下logger和appender的配置细节。

在最前面介绍中提到，root是根logger,所以他两是一回事；只不过root中不能有name和additivity属性，是有一个level。

appender是一个日志打印的组件，这里组件里面定义了打印过滤的条件、打印输出方式、滚动策略、编码方式、打印格式等等。但是它仅仅是一个打印组件，如果我们不使用一个logger或者root的appender-ref指定某个具体的appender时，它就没有什么意义。

因此appender让我们的应用知道怎么打、打印到哪里、打印成什么样；而logger则是告诉应用哪些可以这么打。例如某个类下的日志可以使用这个appender打印或者某个包下的日志可以这么打印。

## appender 配置详解

这里以上面案例中的名为GLMAPPER-LOGGERONE的appender说明：
```xml
<appender name="GLMAPPER-LOGGERONE"
    class="ch.qos.logback.core.rolling.RollingFileAppender">
    <append>true</append>
    <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
        <level>${logging.level}</level>
    </filter>
    <file>
        ${logging.path}/glmapper-spring-boot/glmapper-loggerone.log
    </file>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
        <FileNamePattern>${logging.path}/glmapper-spring-boot/glmapper-loggerone.log.%d{yyyy-MM-dd}</FileNamePattern>
        <MaxHistory>30</MaxHistory>
    </rollingPolicy>
    <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
        <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
        <charset>UTF-8</charset>
    </encoder>
</appender>
```

appender 有两个属性 name和class;name指定appender名称，class指定appender的全限定名。上面声明的是名为GLMAPPER-LOGGERONE，class为ch.qos.logback.core.rolling.RollingFileAppender的一个appender。

1. appender 的种类
* ConsoleAppender：把日志添加到控制台
* FileAppender：把日志添加到文件
* RollingFileAppender：滚动记录文件，先将日志记录到指定文件，当符合某个条件时，将日志记录到其他文件。它是FileAppender的子类

2. append 子标签
```xml
<append>true</append>
```

3. filter 子标签
在简介中提到了filter；作用就是上面说的。可以为appender 添加一个或多个过滤器，可以用任意条件对日志进行过滤。appender 有多个过滤器时，按照配置顺序执行。

4. ThresholdFilter
临界值过滤器，过滤掉低于指定临界值的日志。当日志级别等于或高于临界值时，过滤器返回NEUTRAL；当日志级别低于临界值时，日志会被拒绝。
```xml
<filter class="ch.qos.logback.classic.filter.ThresholdFilter">
    <level>INFO</level>
</filter>
```

5. LevelFilter
级别过滤器，根据日志级别进行过滤。如果日志级别等于配置级别，过滤器会根据onMath(用于配置符合过滤条件的操作) 和 onMismatch(用于配置不符合过滤条件的操作)接收或拒绝日志。
```xml
<filter class="ch.qos.logback.classic.filter.LevelFilter">   
  <level>INFO</level>   
  <onMatch>ACCEPT</onMatch>   
  <onMismatch>DENY</onMismatch>   
</filter> 
```

关于NEUTRAL、ACCEPT、DENY 见上文简介中关于filter的介绍。

6. file 子标签
file 标签用于指定被写入的文件名，可以是相对目录，也可以是绝对目录，如果上级目录不存在会自动创建，没有默认值。
```xml
<file>
    ${logging.path}/glmapper-spring-boot/glmapper-loggerone.log
</file>
```

这个表示当前appender将会将日志写入到${logging.path}/glmapper-spring-boot/glmapper-loggerone.log这个目录下。

7. rollingPolicy 子标签
这个子标签用来描述滚动策略的。这个只有appender的class是RollingFileAppender时才需要配置。这个也会涉及文件的移动和重命名（a.log->a.log.2018.07.22）。

8. TimeBasedRollingPolicy
最常用的滚动策略，它根据时间来制定滚动策略，既负责滚动也负责出发滚动。这个下面又包括了两个属性：
* FileNamePattern
* maxHistory
```xml
<rollingPolicy 
    class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
    <!--日志文件输出的文件名:按天回滚 daily -->
    <FileNamePattern>
        ${logging.path}/glmapper-spring-boot/glmapper-loggerone.log.%d{yyyy-MM-dd}
    </FileNamePattern>
    <!--日志文件保留天数-->
    <MaxHistory>30</MaxHistory>
</rollingPolicy>
```

上面的这段配置表明每天生成一个日志文件，保存30天的日志文件

9. FixedWindowRollingPolicy
根据固定窗口算法重命名文件的滚动策略。

10. encoder 子标签
对记录事件进行格式化。它干了两件事：
* 把日志信息转换成字节数组
* 把字节数组写入到输出流
```xml
<encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
    <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50}
    - %msg%n</pattern>
    <charset>UTF-8</charset>
</encoder>
```

目前encoder只有PatternLayoutEncoder一种类型。

11. 定义一个只打印error级别日志的appcener
```xml
 <!-- 错误日志 appender ： 按照每天生成日志文件 -->
<appender name="ERROR-APPENDER" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <append>true</append>
    <!-- 过滤器，只记录 error 级别的日志 -->
    <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
        <level>error</level>
    </filter>
    <!-- 日志名称 -->
    <file>${logging.path}/glmapper-spring-boot/glmapper-error.log</file>
    <!-- 每天生成一个日志文件，保存30天的日志文件 -->
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
        <!--日志文件输出的文件名:按天回滚 daily -->
        <FileNamePattern>${logging.path}/glmapper-spring-boot/glmapper-error.log.%d{yyyy-MM-dd}</FileNamePattern>
        <!--日志文件保留天数-->
        <MaxHistory>30</MaxHistory>
    </rollingPolicy>
    <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
        <!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度%msg：日志消息，%n是换行符-->
        <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
        <!-- 编码 -->
        <charset>UTF-8</charset>
    </encoder>
</appender>
```

12. 定义一个输出到控制台的appender
```xml
<!-- 默认的控制台日志输出，一般生产环境都是后台启动，这个没太大作用 -->
<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
        <Pattern>%d{HH:mm:ss.SSS} %-5level %logger{80} - %msg%n</Pattern>
    </encoder>
</appender>
```

## logger 配置详解
```xml
<logger name="com.glmapper.spring.boot.controller"
        level="${logging.level}" additivity="false">
    <appender-ref ref="GLMAPPER-LOGGERONE" />
</logger>
```

上面的这个配置文件描述的是：com.glmapper.spring.boot.controller这个包下的${logging.level}级别的日志将会使用GLMAPPER-LOGGERONE来打印。logger有三个属性和一个子标签：

* name:用来指定受此logger约束的某一个包或者具体的某一个类。
* level:用来设置打印级别（TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF），还有一个值INHERITED或者同义词NULL，代表强制执行上级的级别。如果没有设置此属性，那么当前logger将会继承上级的级别。
* addtivity:用来描述是否向上级logger传递打印信息。默认是true。

appender-ref则是用来指定具体appender的。

## 不同日志隔离打印案例
在前面的例子中我们有三种appender,一个是指定包约束的，一个是控制error级别的，一个是控制台的。然后这小节我们就来实现下不同日志打印到不同的log文件中。

1. 根据包进行日志文件隔离
这个例子里我们将com.glmapper.spring.boot.controller中的日志输出到glmapper-controller.log；将com.glmapper.spring.boot.service中的日志输出到glmapper-service.log。

```xml
<!--打印日志到glmapper-service.log的appender-->
<appender name="GLMAPPER-SERVICE"
          class="ch.qos.logback.core.rolling.RollingFileAppender">
    <append>true</append>
    <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
        <level>${logging.level}</level>
    </filter>
    <file>
        ${logging.path}/glmapper-spring-boot/glmapper-service.log
    </file>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
        <FileNamePattern>${logging.path}/glmapper-spring-boot/glmapper-service.log.%d{yyyy-MM-dd}</FileNamePattern>
        <MaxHistory>30</MaxHistory>
    </rollingPolicy>
    <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
        <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
        <charset>UTF-8</charset>
    </encoder>
</appender>

<!--打印日志到glmapper-controller.log的appender-->
<appender name="GLMAPPER-CONTROLLER"
          class="ch.qos.logback.core.rolling.RollingFileAppender">
    <append>true</append>
    <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
        <level>${logging.level}</level>
    </filter>
    <file>
        ${logging.path}/glmapper-spring-boot/glmapper-controller.log
    </file>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
        <FileNamePattern>${logging.path}/glmapper-spring-boot/glmapper-controller.log.%d{yyyy-MM-dd}</FileNamePattern>
        <MaxHistory>30</MaxHistory>
    </rollingPolicy>
    <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
        <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
        <charset>UTF-8</charset>
    </encoder>
</appender>

<!--此logger约束将.controller包下的日志输出到GLMAPPER-CONTROLLER，错误日志输出到GERROR-APPENDE；GERROR-APPENDE见上面-->
<logger name="com.glmapper.spring.boot.controller" level="${logging.level}" additivity="false">
    <appender-ref ref="GLMAPPER-CONTROLLER" />
    <appender-ref ref="GERROR-APPENDER" />
</logger>

<!--此logger约束将.service包下的日志输出到GLMAPPER-SERVICE，错误日志输出到GERROR-APPENDE；GERROR-APPENDE见上面-->
<logger name="com.glmapper.spring.boot.service" level="${logging.level}" additivity="false">
    <appender-ref ref="GLMAPPER-SERVICE" />
    <appender-ref ref="GERROR-APPENDER" />
</logger>
```

满足我们的预期，但是这里有个小问题。在info日志里出现了error,当然这是正常的。假如我们不想在info里面出现error怎么办呢？很简单，我们以APPENDER-SERVICE为例，将filter过滤器进行修改：

将下面的：
```xml
<filter class="ch.qos.logback.classic.filter.ThresholdFilter">
    <level>${logging.level}</level>
</filter>
```

修改为：
```xml
<filter class="ch.qos.logback.classic.filter.LevelFilter">
    <level>ERROR</level>
    <!-- 如果命中就禁止这条日志 -->
    <onMatch>DENY</onMatch>  
    <!-- 如果没有命中就使用这条规则 -->
    <onMismatch>ACCEPT</onMismatch>  
</filter>
```

这里同时要注意的是，在logger中level需要设置为info级别。

2. 根据类进行日志文件隔离
这个其实也是和上面那个差不过，只不过粒度更细一点，一般情况下比如说我们有个定时任务类需要单独来记录其日志信息，这样我们就可以考虑使用基于类维度来约束打印。
```xml
<!--特殊功能单独appender 例如调度类的日志-->
<appender name="SCHEDULERTASKLOCK-APPENDER" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <append>true</append>
    <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
        <level>${logging.level}</level>
    </filter>
    <file>${logging.path}/glmapper-spring-boot/scheduler-task-lock.log</file>
    <!-- 每天生成一个日志文件，保存30天的日志文件 -->
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
        <!--日志文件输出的文件名:按天回滚 daily -->
        <FileNamePattern>${logging.path}/glmapper-spring-boot/scheduler-task-lock.log.%d{yyyy-MM-dd}</FileNamePattern>
        <!--日志文件保留天数-->
        <MaxHistory>30</MaxHistory>
    </rollingPolicy>
    <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
        <!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度%msg：日志消息，%n是换行符-->
        <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
        <!-- 编码 -->
        <charset>UTF-8</charset>
    </encoder>
</appender>

<!--这里指定到了具体的某一个类-->
<logger name="com.glmapper.spring.boot.task.TestLogTask" level="${logging.level}" additivity="true">
        <appender-ref ref="SCHEDULERTASKLOCK-APPENDER" />
        <appender-ref ref="ERROR-APPENDER" />
    </logger>
```

3. 根据自定义 logger 的 name 进行日志文件隔离
logger的name除了类、包等约束之外，当然还可以这样来玩。。。

* 在进行案例之前，这里先把前面案例中logger声明的代码贴一下，以作对比,以TestLogTask类中的日志为例：
```java
 private static final Logger LOGGER =
 LoggerFactory.getLogger(TestLogTask.class);
```

* 在getLogger中我们是将当前对象的class作为参数的，这个是为了打印时获取其全限定名的
```java
1-2018-07-21 11:15:42.003 [pool-1-thread-1] 
2-INFO  
3-com.glmapper.spring.boot.task.TestLogTask -
4-com.glmapper.spring.boot.task:info
```

* 业务类定义
我们同样是service包下定义一个类TestLogNameServiceImpl

```java
package com.glmapper.spring.boot.service;

@Service("testLogNameService")
public class TestLogNameServiceImpl implements TestLogNameService {

    private static final Logger LOGGER =
    LoggerFactory.getLogger("GLMAPPER-TEST-LOG");

    @Override
    public void print() {
        LOGGER.info("GLMAPPER-TEST-LOG:this is special logger-----info");
        LOGGER.error("GLMAPPER-TEST-LOG:this is special logger-------error");
    }
}
```

* appender和logger配置
```xml
<appender name="ROOT-APPENDER" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <append>true</append>
    <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
        <level>${logging.level}</level>
    </filter>
    <file>${logging.path}/glmapper-spring-boot/glmapper-test.log</file>
    <!-- 每天生成一个日志文件，保存30天的日志文件 -->
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
        <!--日志文件输出的文件名:按天回滚 daily -->
        <FileNamePattern>${logging.path}/glmapper-spring-boot/glmapper-test.log.%d{yyyy-MM-dd}
        </FileNamePattern>
        <!--日志文件保留天数-->
        <MaxHistory>30</MaxHistory>
    </rollingPolicy>
    <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
        <!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度%msg：日志消息，%n是换行符-->
        <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
        <!-- 编码 -->
        <charset>UTF-8</charset>
    </encoder>
</appender>

<!--这里的name和业务类中的getLogger中的字符串是一样的-->
<logger name="GLMAPPER-TEST-LOG" level="${logging.level}" additivity="true">
        <appender-ref ref="ROOT-APPENDER" />
        <appender-ref ref="ERROR-APPENDER" />
    </logger>
```

我们这个预期的是TestLogNameServiceImpl中的日志不打印到glmapper-service.log中，而是打印到glmapper-test.log中。

## 如何使用logback打印mybatis的sql语句
这个还是比较坑的。为什么。看下这个：

```xml
<settings>
    <setting name="logImpl" value="slf4j" />
</settings>
```

在mybatis-configration.xml中，我们通过这样一个配置项来关联到具体的日志组件。但是logImpl的实现中是没有logback的。那么怎么办呢？这里只能通过slf4j的方式桥接到logback。

然后在我们的logback-spring.xml中进行如下配置：

```xml
 <!-- 将sql语句输出到具体的日志文件中 -->
<logger name="com.alipay.sofa.cloudplatform.common.dao" level="${logging.sql.level}" additivity="false">
    <appender-ref ref="SQL-APPENDER"/>
</logger>
```

这里有几个点需要注意的。首先是${logging.sql.level}这个必须是debug，这个是由mybatis本身实现决定的。而这里的name设定的com.alipay.sofa.cloudplatform.common.dao值就是我们dao接口的包路径。
网上看了一个比较典型的案例，这种方式只能输出到控制台，并不能将文件输出到日志文件；它是根据内部的一个实现机制偷了个懒。mybatis用logback日志不显示sql的解决办法。


# 参考

https://juejin.im/post/5b51f85c5188251af91a7525