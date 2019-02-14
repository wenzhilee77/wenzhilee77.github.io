---
layout: post
title:  "SpringBoot整合Lettuce Redis"
categories: Redis SpringBoot
tags:  SpringBoot Lettuce Redis 线程安全  
author: wenzhilee77
---

* content
{:toc}


## Lettuce

Lettuce和Jedis的都是连接Redis Server的客户端程序。Jedis在实现上是直连redis server，多线程环境下非线程安全，除非使用连接池，为每个Jedis实例增加物理连接。Lettuce基于Netty的连接实例（StatefulRedisConnection），可以在多个线程间并发访问，且线程安全，满足多线程环境下的并发访问，同时它是可伸缩的设计，一个连接实例不够的情况也可以按需增加连接实例。

##  导入依赖

Spring Boot1.5的版本默认采用的连接池技术是jedis，2.0以上版本默认连接池是lettuce。
spring2.0集成redis所需common-pool2。
Redis对象序列化器jackson-databind。
```java
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>

        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
            <version>2.6.0</version>
        </dependency>

        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.9.8</version>
        </dependency>
    </dependencies>
```

## 属性配置
```java
server:
  port: 8088

spring:
  redis:
    host: 127.0.0.1
    port: 6379
    database: 0
    timeout: 10000
    lettuce:
      pool:
        max-active: 8
        max-idle: 8
        max-wait: -1
        min-idle: 0
```

## 自定义Template

```java
/**
 * Redis Configuration
 *
 * @author didi
 * @date 2019/02/13
 */

@Configuration
@AutoConfigureAfter(RedisAutoConfiguration.class)
public class RedisConfiguration
{
    @Bean
    public RedisTemplate<String, Serializable> redisTemplate(LettuceConnectionFactory redisConnectionFactory)
    {
        RedisTemplate<String, Serializable> template = new RedisTemplate<>();
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }
}
```

## 实体类

```java
/**
 * Demo class
 *
 * @author didi
 * @date 2019/02/13
 */
public class User implements Serializable
{
    private long id;
    private String userName;
    private String password;


    public User()
    {

    }

    public User(String userName, String password)
    {
        this.userName = userName;
        this.password = password;
    }

    public User(long id, String userName, String password)
    {
        this.id = id;
        this.userName = userName;
        this.password = password;
    }

    public long getId()
    {
        return id;
    }

    public void setId(long id)
    {
        this.id = id;
    }

    public String getUserName()
    {
        return userName;
    }

    public void setUserName(String userName)
    {
        this.userName = userName;
    }

    public String getPassword()
    {
        return password;
    }

    public void setPassword(String password)
    {
        this.password = password;
    }

    @Override
    public String toString()
    {
        return "User{" + "id=" + id + ", userName='" + userName + '\'' + ", password='" + password + '\'' + '}';
    }
}
```

##  测试

```java
public class RedisDemoApplicationTests
{
    private static final Logger log = LoggerFactory.getLogger(RedisDemoApplicationTests.class);

    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    @Autowired
    private RedisTemplate<String, Serializable> redisTemplate;

    @Test
    public void contextLoads()
    {
        ExecutorService executorService = Executors.newFixedThreadPool(1000);
        IntStream.range(0, 1000).forEach(i ->
                executorService.execute(() -> stringRedisTemplate.opsForValue().increment("kk", 1))
        );
        stringRedisTemplate.opsForValue().set("k1", "v1");
        final String k1 = stringRedisTemplate.opsForValue().get("k1");
        log.info("[字符缓存结果] - [{}]", k1);

        String key = "battcn:user:1";
        redisTemplate.opsForValue().set(key, new User(1L, "u1", "pa"));

        // TODO 对应 String（字符串）
        final User user = (User) redisTemplate.opsForValue().get(key);
        log.info("[对象缓存结果] - [{}]", user);
    }
}
```

引用：

https://blog.csdn.net/weixin_39723544/article/details/80743074
https://blog.csdn.net/winter_chen001/article/details/80614331