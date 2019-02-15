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

## 导入依赖

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

## 封装redis操作

```java
package com.wenzhli.redis.demo.util;


import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.StringRedisTemplate;

import java.io.Serializable;
import java.util.*;
import java.util.concurrent.TimeUnit;

/**
 * redis工具类
 *
 * @author wenzhilee77
 */
@SuppressWarnings("unchecked")
public class RedisUtil
{
    private static final Logger LOG = LoggerFactory.getLogger(RedisUtil.class);

    @Autowired
    private static RedisTemplate<String, Object> redisTemplate;

    private static String CACHE_PREFIX;

    private static boolean CACHE_CLOSED;

    private RedisUtil()
    {

    }

    @SuppressWarnings("rawtypes")
    private static boolean isEmpty(Object obj)
    {
        if (obj == null)
        {
            return true;
        }
        else if (obj instanceof String)
        {
            String str = obj.toString();
            return "".equals(str.trim());
        }
        else if (obj instanceof List)
        {
            List<Object> list = (List<Object>) obj;
            return list.isEmpty();
        }
        else if (obj instanceof Map)
        {
            Map map = (Map) obj;
            return map.isEmpty();
        }
        else if (obj instanceof Set)
        {
            Set set = (Set) obj;
            return set.isEmpty();
        }
        else if (obj instanceof Object[])
        {
            Object[] objs = (Object[]) obj;
            return objs.length <= 0;
        }
        return false;
    }

    /**
     * 构建缓存key值
     *
     * @param key 缓存key
     */
    private static String buildKey(String key)
    {
        if (CACHE_PREFIX == null || "".equals(CACHE_PREFIX))
        {
            return key;
        }
        return CACHE_PREFIX + ":" + key;
    }

    /**
     * 返回缓存的前缀
     *
     * @return CACHE_PREFIX_FLAG
     */
    public static String getCachePrefix()
    {
        return CACHE_PREFIX;
    }

    /**
     * 设置缓存的前缀
     *
     * @param cachePrefix
     */
    public static void setCachePrefix(String cachePrefix)
    {
        if (cachePrefix != null && !"".equals(cachePrefix.trim()))
        {
            CACHE_PREFIX = cachePrefix.trim();
        }
    }

    /**
     * 关闭缓存
     *
     * @return true:成功 false:失败
     */
    public static boolean close()
    {
        LOG.debug(" cache closed ! ");
        CACHE_CLOSED = true;
        return true;
    }

    /**
     * 打开缓存
     *
     * @return true:存在 false:不存在
     */
    public static boolean openCache()
    {
        CACHE_CLOSED = false;
        return true;
    }

    /**
     * 检查缓存是否开启
     *
     * @return true:已关闭 false:已开启
     */
    public static boolean isClose()
    {
        return CACHE_CLOSED;
    }

    /**
     * 判断key值是否存在
     *
     * @param key 缓存的key
     * @return true:存在 false:不存在
     */
    public static boolean hasKey(String key)
    {
        LOG.debug(" hasKey key :{}", key);
        try
        {
            if (isClose() || isEmpty(key))
            {
                return false;
            }
            key = buildKey(key);
            return redisTemplate.hasKey(key);
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return false;
    }

    /**
     * 匹配符合正则的key
     *
     * @param patternKey
     * @return key的集合
     */
    public static Set<String> keys(String patternKey)
    {
        LOG.debug(" keys key :{}", patternKey);
        try
        {
            if (isClose() || isEmpty(patternKey))
            {
                return Collections.emptySet();
            }
            return redisTemplate.keys(patternKey);
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return Collections.emptySet();
    }

    /**
     * 根据key删除缓存
     *
     * @param key
     * @return true:成功 false:失败
     */
    public static boolean del(String... key)
    {
        LOG.debug(" delete key :{}", key.toString());
        if (isClose() || isEmpty(key))
        {
            return false;
        }

        try
        {
            Set<String> keySet = new HashSet<>();
            for (String str : key)
            {
                keySet.add(buildKey(str));
            }
            redisTemplate.delete(keySet);
            return true;
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
            return false;
        }
    }

    /**
     * 根据key删除缓存
     *
     * @param key
     * @return true:成功 false:失败
     */
    public static boolean delPattern(String key)
    {
        LOG.debug(" delete Pattern keys :{}", key);
        try
        {
            if (isClose() || isEmpty(key))
            {
                return false;
            }
            key = buildKey(key);
            redisTemplate.delete(redisTemplate.keys(key));
            return true;
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return false;
    }

    /**
     * 删除一组key值
     *
     * @param keys
     * @return true:成功 false:失败
     */
    public static boolean del(Set<String> keys)
    {
        LOG.debug(" delete keys :{}", keys.toString());
        try
        {
            if (isClose() || isEmpty(keys))
            {
                return false;
            }
            Set<String> keySet = new HashSet<>();
            for (String str : keys)
            {
                keySet.add(buildKey(str));
            }
            redisTemplate.delete(keySet);
            return true;
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return false;
    }

    /**
     * 设置过期时间
     *
     * @param key     缓存key
     * @param seconds 过期秒数
     * @return true:成功 false:失败
     */
    public static boolean setExp(String key, long seconds)
    {
        LOG.debug(" setExp key :{}, seconds: {}", key, seconds);
        try
        {
            if (isClose() || isEmpty(key) || seconds <= 0)
            {
                return false;
            }
            key = buildKey(key);
            return redisTemplate.expire(key, seconds, TimeUnit.SECONDS);
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return false;
    }

    /**
     * 查询过期时间
     *
     * @param key 缓存key
     * @return 秒数
     */
    public static Long getExpire(String key)
    {
        LOG.debug(" getExpire key :{}", key);
        try
        {
            if (isClose() || isEmpty(key))
            {
                return 0L;
            }
            key = buildKey(key);
            return redisTemplate.getExpire(key, TimeUnit.SECONDS);
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return 0L;
    }

    /**
     * 缓存存入key-value
     *
     * @param key   缓存键
     * @param value 缓存值
     * @return true:成功 false:失败
     */
    public static boolean setString(String key, String value)
    {
        LOG.debug(" setString key :{}, value: {}", key, value);
        try
        {
            if (isClose() || isEmpty(key) || isEmpty(value))
            {
                return false;
            }
            key = buildKey(key);
            redisTemplate.opsForValue().set(key, value);
            return true;
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return false;
    }

    /**
     * 缓存存入key-value
     *
     * @param key     缓存键
     * @param value   缓存值
     * @param seconds 秒数
     * @return true:成功 false:失败
     */
    public static boolean setString(String key, String value, long seconds)
    {
        LOG.debug(" setString key :{}, value: {}, timeout:{}", key, value, seconds);
        try
        {
            if (isClose() || isEmpty(key) || isEmpty(value))
            {
                return false;
            }
            key = buildKey(key);
            redisTemplate.opsForValue().set(key, value, seconds, TimeUnit.SECONDS);
            return true;
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return false;
    }

    /**
     * 根据key取出String value
     *
     * @param key 缓存key值
     * @return String    缓存的String
     */
    public static String getString(String key)
    {
        LOG.debug(" getString key :{}", key);
        try
        {
            if (isClose() || isEmpty(key))
            {
                return null;
            }
            key = buildKey(key);
            return (String) redisTemplate.opsForValue().get(key);
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return null;
    }

    /**
     * 去的缓存中的最大值并+1
     *
     * @param key 缓存key值
     * @return long    缓存中的最大值+1
     */
    public static long incr(String key)
    {
        LOG.debug(" incr key :{}", key);
        try
        {
            if (isClose() || isEmpty(key))
            {
                return 0;
            }
            key = buildKey(key);
            return redisTemplate.opsForValue().increment(key, 1);
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return 0;
    }

    /**
     * 缓存中存入序列化的Object对象
     *
     * @param key 缓存key
     * @param obj 存入的序列化对象
     * @return true:成功 false:失败
     */
    public static boolean setObj(String key, Object obj)
    {
        LOG.debug(" set key :{}, value:{}", key, obj);
        try
        {
            if (isClose() || isEmpty(key) || isEmpty(obj))
            {
                return false;
            }
            key = buildKey(key);
            redisTemplate.opsForValue().set(key, obj);
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return false;
    }

    /**
     * 缓存中存入序列化的Object对象
     *
     * @param key 缓存key
     * @param obj 存入的序列化对象
     * @return true:成功 false:失败
     */
    public static boolean setObj(String key, Object obj, long seconds)
    {
        LOG.debug(" set key :{}, value:{}, seconds:{}", key, obj, seconds);
        try
        {
            if (isClose() || isEmpty(key) || isEmpty(obj))
            {
                return false;
            }
            key = buildKey(key);
            redisTemplate.opsForValue().set(key, obj);
            if (seconds > 0)
            {
                redisTemplate.expire(key, seconds, TimeUnit.SECONDS);
            }
            return true;
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return false;
    }

    /**
     * 取出缓存中存储的序列化对象
     *
     * @param key   缓存key
     * @param clazz 对象类
     * @return <T>	序列化对象
     */
    public static <T> T getObj(String key, Class<T> clazz)
    {
        LOG.debug(" get key :{}", key);
        try
        {
            if (isClose() || isEmpty(key))
            {
                return null;
            }
            key = buildKey(key);
            return (T) redisTemplate.opsForValue().get(key);
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return null;
    }

    /**
     * 存入Map数组
     *
     * @param <T>
     * @param key 缓存key
     * @param map 缓存map
     * @return true:成功 false:失败
     */
    public static <T> boolean setMap(String key, Map<String, T> map)
    {
        try
        {
            if (isClose() || isEmpty(key) || isEmpty(map))
            {
                return false;
            }
            key = buildKey(key);
            redisTemplate.opsForHash().putAll(key, map);
            return true;
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return false;
    }

    /**
     * 取出缓存的map
     *
     * @param key 缓存key
     * @return map    缓存的map
     */
    @SuppressWarnings("rawtypes")
    public static Map getMap(String key)
    {
        LOG.debug(" getMap key :{}", key);
        try
        {
            if (isClose() || isEmpty(key))
            {
                return null;
            }
            key = buildKey(key);
            return redisTemplate.opsForHash().entries(key);
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return null;
    }

    /**
     * 查询缓存的map的集合大小
     *
     * @param key 缓存key
     * @return int    缓存map的集合大小
     */
    public static long getMapSize(String key)
    {
        LOG.debug(" getMap key :{}", key);
        try
        {
            if (isClose() || isEmpty(key))
            {
                return 0;
            }
            key = buildKey(key);
            return redisTemplate.opsForHash().size(key);
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return 0;
    }


    /**
     * 根据key以及hashKey取出对应的Object对象
     *
     * @param key     缓存key
     * @param hashKey 对应map的key
     * @return object    map中的对象
     */
    public static Object getMapKey(String key, String hashKey)
    {
        LOG.debug(" getMapkey :{}, hashKey:{}", key, hashKey);
        try
        {
            if (isClose() || isEmpty(key) || isEmpty(hashKey))
            {
                return null;
            }
            key = buildKey(key);
            return redisTemplate.opsForHash().get(key, hashKey);
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return null;
    }

    /**
     * 取出缓存中map的所有key值
     *
     * @param key 缓存key
     * @return Set<String> map的key值合集
     */
    public static Set<Object> getMapKeys(String key)
    {
        LOG.debug(" getMapKeys key :{}", key);
        try
        {
            if (isClose() || isEmpty(key))
            {
                return null;
            }
            key = buildKey(key);
            return redisTemplate.opsForHash().keys(key);
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return null;
    }

    /**
     * 删除map中指定的key值
     *
     * @param key     缓存key
     * @param hashKey map中指定的hashKey
     * @return true:成功 false:失败
     */
    public static boolean delMapKey(String key, String hashKey)
    {
        LOG.debug(" delMapKey key :{}, hashKey:{}", key, hashKey);
        try
        {
            if (isClose() || isEmpty(key) || isEmpty(hashKey))
            {
                return false;
            }
            key = buildKey(key);
            redisTemplate.opsForHash().delete(key, hashKey);
            return true;
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return false;
    }

    /**
     * 存入Map数组
     *
     * @param <T>
     * @param key     缓存key
     * @param map     缓存map
     * @param seconds 秒数
     * @return true:成功 false:失败
     */
    public static <T> boolean setMapExp(String key, Map<String, T> map, long seconds)
    {
        LOG.debug(" setMapExp key :{}, value: {}, seconds:{}", key, map, seconds);
        if (isClose() || isEmpty(key) || isEmpty(map))
        {
            return false;
        }
        try
        {
            key = buildKey(key);
            redisTemplate.opsForHash().putAll(key, map);
            redisTemplate.expire(key, seconds, TimeUnit.SECONDS);
            return true;
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return false;
    }

    /**
     * map中加入新的key
     *
     * @param <T>
     * @param key     缓存key
     * @param hashKey map的Key值
     * @param value   map的value值
     * @return true:成功 false:失败
     */
    public static <T> boolean addMap(String key, String hashKey, T value)
    {
        LOG.debug(" addMap key :{}, hashKey: {}, value:{}", key, hashKey, value);
        if (isClose() || isEmpty(key) || isEmpty(hashKey) || isEmpty(value))
        {
            return false;
        }
        try
        {
            key = buildKey(key);
            redisTemplate.opsForHash().put(key, hashKey, value);
            return true;
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return false;
    }

    /**
     * 缓存存入List
     *
     * @param <T>
     * @param key  缓存key
     * @param list 缓存List
     * @return true:成功 false:失败
     */
    public static <T> boolean setList(String key, List<T> list)
    {
        LOG.debug(" setList key :{}, list: {}", key, list);
        if (isClose() || isEmpty(key) || isEmpty(list))
        {
            return false;
        }
        try
        {
            key = buildKey(key);
            redisTemplate.opsForList().leftPushAll(key, list.toArray());
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return false;
    }

    /**
     * 根据key值取出对应的list合集
     *
     * @param key 缓存key
     * @return List<Object> 缓存中对应的list合集
     */
    public static <V> List<V> getList(String key)
    {
        LOG.debug(" getList key :{}", key);
        try
        {
            if (isClose() || isEmpty(key))
            {
                return null;
            }
            key = buildKey(key);
            return (List<V>) redisTemplate.opsForList().range(key, 0, -1);
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return null;
    }

    /**
     * 根据key值截取对应的list合集
     *
     * @param key   缓存key
     * @param start 开始位置
     * @param end   结束位置
     * @return
     */
    public static void trimList(String key, int start, int end)
    {
        LOG.debug(" trimList key :{}", key);
        try
        {
            if (isClose() || isEmpty(key))
            {
                return;
            }
            key = buildKey(key);
            redisTemplate.opsForList().trim(key, start, end);
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
    }

    /**
     * 取出list合集中指定位置的对象
     *
     * @param key   缓存key
     * @param index 索引位置
     * @return Object    list指定索引位置的对象
     */
    public static Object getIndexList(String key, int index)
    {
        LOG.debug(" getIndexList key :{}, index:{}", key, index);
        try
        {
            if (isClose() || isEmpty(key) || index < 0)
            {
                return null;
            }
            key = buildKey(key);
            return redisTemplate.opsForList().index(key, index);
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return null;
    }

    /**
     * Object存入List
     *
     * @param key   缓存key
     * @param value List中的值
     * @return true:成功 false:失败
     */
    public static boolean addList(String key, Object value)
    {
        LOG.debug(" addList key :{}, value:{}", key, value);
        try
        {
            if (isClose() || isEmpty(key) || isEmpty(value))
            {
                return false;
            }
            key = buildKey(key);
            redisTemplate.opsForList().leftPush(key, value);
            return true;
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return false;
    }

    /**
     * 缓存存入List
     *
     * @param <T>
     * @param key     缓存key
     * @param list    缓存List
     * @param seconds 秒数
     * @return true:成功 false:失败
     */
    public static <T> boolean setList(String key, List<T> list, long seconds)
    {
        LOG.debug(" setList key :{}, value:{}, seconds:{}", key, list, seconds);
        try
        {
            if (isClose() || isEmpty(key) || isEmpty(list))
            {
                return false;
            }
            key = buildKey(key);
            redisTemplate.opsForList().leftPushAll(key, list.toArray());
            if (seconds > 0)
            {
                redisTemplate.expire(key, seconds, TimeUnit.SECONDS);
            }
            return true;
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return false;
    }

    /**
     * set集合存入缓存
     *
     * @param <T>
     * @param key 缓存key
     * @param set 缓存set集合
     * @return true:成功 false:失败
     */
    public static <T> boolean setSet(String key, Set<T> set)
    {
        LOG.debug(" setSet key :{}, value:{}", key, set);
        try
        {
            if (isClose() || isEmpty(key) || isEmpty(set))
            {
                return false;
            }
            key = buildKey(key);
            redisTemplate.opsForSet().add(key, set.toArray());
            return true;
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return false;
    }

    /**
     * set集合中增加value
     *
     * @param key   缓存key
     * @param value 增加的value
     * @return true:成功 false:失败
     */
    public static boolean addSet(String key, Object value)
    {
        LOG.debug(" addSet key :{}, value:{}", key, value);
        try
        {
            if (isClose() || isEmpty(key) || isEmpty(value))
            {
                return false;
            }
            key = buildKey(key);
            redisTemplate.opsForSet().add(key, value);
            return true;
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return false;
    }

    /**
     * set集合存入缓存
     *
     * @param <T>
     * @param key     缓存key
     * @param set     缓存set集合
     * @param seconds 秒数
     * @return true:成功 false:失败
     */
    public static <T> boolean setSet(String key, Set<T> set, long seconds)
    {
        LOG.debug(" setSet key :{}, value:{}, seconds:{}", key, set, seconds);
        try
        {
            if (isClose() || isEmpty(key) || isEmpty(set))
            {
                return false;
            }
            key = buildKey(key);
            redisTemplate.opsForSet().add(key, set.toArray());
            if (seconds > 0)
            {
                redisTemplate.expire(key, seconds, TimeUnit.SECONDS);
            }
            return true;
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return false;
    }

    /**
     * 取出缓存中对应的set合集
     *
     * @param <T>
     * @param key 缓存key
     * @return Set<Object> 缓存中的set合集
     */
    public static <T> Set<T> getSet(String key)
    {
        LOG.debug(" getSet key :{}", key);
        try
        {
            if (isClose() || isEmpty(key))
            {
                return null;
            }
            key = buildKey(key);
            return (Set<T>) redisTemplate.opsForSet().members(key);
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return null;
    }

    /**
     * 有序集合存入数值
     *
     * @param key   缓存key
     * @param value 缓存value
     * @param score 评分
     * @return
     */
    public static boolean addZSet(String key, Object value, double score)
    {
        LOG.debug(" addZSet key :{},value:{}, score:{}", key, value, score);
        try
        {
            if (isClose() || isEmpty(key) || isEmpty(value))
            {
                return false;
            }
            key = buildKey(key);
            return redisTemplate.opsForZSet().add(key, value, score);
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return false;
    }

    /**
     * 从有序集合中删除指定值
     *
     * @param key   缓存key
     * @param value 缓存value
     * @return
     */
    public static boolean removeZSet(String key, Object value)
    {
        LOG.debug(" removeZSet key :{},value:{}", key, value);
        try
        {
            if (isClose() || isEmpty(key) || isEmpty(value))
            {
                return false;
            }
            key = buildKey(key);
            redisTemplate.opsForZSet().remove(key, value);
            return true;
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return false;
    }

    /**
     * 从有序集合中删除指定位置的值
     *
     * @param key   缓存key
     * @param start 起始位置
     * @param end   结束为止
     * @return
     */
    public static boolean removeZSet(String key, long start, long end)
    {
        LOG.debug(" removeZSet key :{},start:{}, end:{}", key, start, end);
        try
        {
            if (isClose() || isEmpty(key))
            {
                return false;
            }
            key = buildKey(key);
            redisTemplate.opsForZSet().removeRange(key, start, end);
            return true;
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return false;
    }

    /**
     * 从有序集合中获取指定位置的值
     *
     * @param key   缓存key
     * @param start 起始位置
     * @param end   结束为止
     * @return
     */
    public static <T> Set<T> getZSet(String key, long start, long end)
    {
        LOG.debug(" getZSet key :{},start:{}, end:{}", key, start, end);
        try
        {
            if (isClose() || isEmpty(key))
            {
                return Collections.emptySet();
            }
            key = buildKey(key);
            return (Set<T>) redisTemplate.opsForZSet().range(key, start, end);
        }
        catch (Exception e)
        {
            LOG.error(e.getMessage(), e);
        }
        return Collections.emptySet();
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
public class User
{
    private long id;
    private String userName;
    private String password;


    public User()
    {

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
    private RedisTemplate<String, Serializable> redisTemplate;

    @Test
    public void contextLoads()
    {
        ExecutorService executorService = Executors.newFixedThreadPool(1000);
        IntStream.range(0, 1000).forEach(i ->
                executorService.execute(() -> stringRedisTemplate.opsForValue().increment("kk", 1))
        );
        redisTemplate.opsForValue().set("k1", "v1");
        final String k1 = redisTemplate.opsForValue().get("k1");
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