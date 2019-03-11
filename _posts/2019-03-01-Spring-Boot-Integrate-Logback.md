---
layout: post
title:  "SpringBoot 整合 logback"
categories: SpringBoot logback
tags:  SpringBoot logback
author: wenzhilee77
---

# application.yml

```
logging:
  config: classpath:logback-spring.xml
```

# logback-spring.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<configuration  scan="true" scanPeriod="10 seconds">

    <contextName>logback</contextName>
    <property name="log.path" value="/Users/didi/log" />
    <property name="CONSOLE_LOG_PATTERN" value="[%-5level][%d{YYYY-MM-dd'T'HH:mm:ss.SSS+0800}][%F:%line] %m%n"/>

    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${CONSOLE_LOG_PATTERN}</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <appender name="INFO_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log.path}/info/log_info.log</file>
        <encoder>
            <pattern>${CONSOLE_LOG_PATTERN}</pattern>
            <charset>UTF-8</charset>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log.path}/info/log-info-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <maxHistory>15</maxHistory>
        </rollingPolicy>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>info</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <appender name="WARN_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log.path}/warn/log_warn.log</file>
        <encoder>
            <pattern>${CONSOLE_LOG_PATTERN}</pattern>
            <charset>UTF-8</charset>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log.path}/warn/log-warn-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <maxHistory>15</maxHistory>
        </rollingPolicy>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>warn</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log.path}/error/log_error.log</file>
        <encoder>
            <pattern>${CONSOLE_LOG_PATTERN}</pattern>
            <charset>UTF-8</charset>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log.path}/error/log-error-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <maxHistory>15</maxHistory>
        </rollingPolicy>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <root level="info">
        <appender-ref ref="STDOUT"/>
    </root>

    <logger name="com.wenzhli.log.format.util.LogAop" level="info">
        <appender-ref ref="INFO_FILE" />
        <appender-ref ref="WARN_FILE" />
        <appender-ref ref="ERROR_FILE" />
    </logger>

</configuration>
```

# LogAop.java

```
package com.wenzhli.log.format.util;

import com.wenzhli.log.format.model.ErrorType;
import com.wenzhli.log.format.model.IpGenerator;
import com.wenzhli.log.format.model.LogEntety;
import com.wenzhli.log.format.model.LogFlowTag;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.Signature;
import org.aspectj.lang.annotation.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import javax.servlet.http.HttpServletRequest;
import java.util.Enumeration;

@Aspect
@Component
@Order(1)
public class LogAop
{
    private Logger logger = LoggerFactory.getLogger(LogAop.class);

    //申明一个切点 里面是 execution表达式
    @Pointcut("execution(public * com.wenzhli.log.format.controller..*(..))")
    private void pointCut(){}

    @Around("pointCut()")
    public Object logFormat(ProceedingJoinPoint joinPoint) throws Throwable
    {
        String traceId = IpGenerator.getTraceId();
        String spanId = IpGenerator.getSpanId();

        long beginTime = System.currentTimeMillis();

        ServletRequestAttributes attributes =
                (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = attributes.getRequest();
        String uri = request.getRequestURI();
        String ip = request.getRemoteAddr();

        Enumeration<String> params = request.getParameterNames();

        StringBuilder sb = new StringBuilder();
        while(params.hasMoreElements())
        {
            String pName = params.nextElement();
            sb.append(request.getParameter(pName) + ",");
        }
        String[] args = sb.toString().split(",");

        Signature signature = joinPoint.getSignature();
        String longSign = signature.toLongString();

        Object result;

        LogEntety logEntety = new LogEntety(traceId, spanId, uri, ip);
        if(args.length != 0)
        {
            logEntety.setArgs(args);
        }

        logger.info(logEntety.toString());

        boolean hasError = false;
        String errMsg = null;
        try
        {
            result = joinPoint.proceed();
        }
        catch (Exception e)
        {
            errMsg = e.getMessage();
            hasError = true;
            throw e;
        }
        finally
        {
            long endTime = System.currentTimeMillis();
            logEntety.setProc_time(endTime - beginTime);
            logEntety.setTag(LogFlowTag.OUT);

            if(hasError)
            {
                logEntety.setErrno(ErrorType.ERROR);
                logEntety.setMsg(errMsg);
                logger.error(logEntety.toString());
            }
            else
            {
                logger.info(logEntety.toString());
            }
        }

        return result;
    }
}
```



