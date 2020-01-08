---
layout: post
title:  Distributed Lock
categories: Distributed Lock Redis Zookeeper
tags:  Distributed Lock Redis Zookeeper
author: wenzhilee77
---

# Redis 实现
```java
public class DistributedLock
{
    @Resource
    private RedisTemplate redisTemplate;

    @Resource
    private DefaultRedisScript<Long> redisScript;

    private static long DEFAULT_SLEEP_TIME = 1000;

    private static long EXPIRED_TIME = 60;

    private static final Long RELEASE_SUCCESS = 1L;

    public boolean tryLock(String key, String value)
    {
        log.info("The params of 'tryLock' are key: {}, value: {}", key, value);
        return redisTemplate.opsForValue().setIfAbsent(key, value, EXPIRED_TIME, TimeUnit.SECONDS);
    }

    public void lock(String key, String value) throws InterruptedException
    {
        log.info("The params of 'lock' are key: {}, value: {}", key, value);
        for (; ; )
        {
            if (redisTemplate.opsForValue().setIfAbsent(key, value, EXPIRED_TIME, TimeUnit.SECONDS))
            {
                break;
            }
            Thread.sleep(DEFAULT_SLEEP_TIME);
        }
    }

    public boolean lock(String key, String value, int blockTime) throws InterruptedException
    {
        log.info("The params of 'lock' are key: {}, value: {}, blockTime: {}", key, value, blockTime);
        while (blockTime >= 0)
        {
            if (redisTemplate.opsForValue().setIfAbsent(key, value, EXPIRED_TIME, TimeUnit.SECONDS))
            {
                return true;
            }
            blockTime -= DEFAULT_SLEEP_TIME;
            Thread.sleep(DEFAULT_SLEEP_TIME);
        }
        return false;
    }

    public boolean unlock(String key, String value)
    {
        log.info("The params of 'unlock' are key: {}, value: {}", key, value);
        Long result = (Long) redisTemplate.execute(redisScript, Arrays.asList(key, value));
        return RELEASE_SUCCESS.equals(result);
    }

    @Bean
    public DefaultRedisScript<Long> defaultRedisScript()
    {
        DefaultRedisScript<Long> defaultRedisScript = new DefaultRedisScript<>();
        defaultRedisScript.setResultType(Long.class);
        defaultRedisScript.setScriptText("if redis.call('get', KEYS[1]) == KEYS[2] then return redis.call('del', KEYS[1]) else return 0 end");
        return defaultRedisScript;
    }
}
```

# Zookeeper 实现
```java
public class DistributedLockByZookeeper 
{
    private final static String ROOT_PATH_LOCK = "lockRoot";

    private CountDownLatch countDownLatch = new CountDownLatch(1);

    @Autowired
    CuratorFramework curatorFramework;

    public void acquireDistributedLock(String path) 
    {
        String keyPath = "/" + ROOT_PATH_LOCK + "/" + path;
        while (true) 
        {
            try 
            {
                curatorFramework.create()
                        .creatingParentsIfNeeded()
                        .withMode(CreateMode.EPHEMERAL)
                        .withACL(ZooDefs.Ids.OPEN_ACL_UNSAFE)
                        .forPath(keyPath);
                log.info("success to acquire lock for path:{}", keyPath);
                break;
            } 
            catch (Exception e) 
            {
                //抢不到锁，进入此处!
                try 
                {
                    if (countDownLatch.getCount() <= 0) 
                    {
                        countDownLatch = new CountDownLatch(1);
                    }
                    //避免请求获取不到锁，重复的while，浪费CPU资源
                    countDownLatch.await();
                } 
                catch (InterruptedException e1) 
                {
                    e1.printStackTrace();
                }
            }
        }
    }

    public boolean releaseDistributedLock(String path) 
    {
        try 
        {
            String keyPath = "/" + ROOT_PATH_LOCK + "/" + path;
            if (curatorFramework.checkExists().forPath(keyPath) != null) 
            {
                curatorFramework.delete().forPath(keyPath);
            }
        } 
        catch (Exception e) 
        {
            log.error("failed to release lock,{}", e);
            return false;
        }
        return true;
    }


    private void addWatcher(String path) 
    {
        String keyPath;
        if (path.equals(ROOT_PATH_LOCK)) 
        {
            keyPath = "/" + path;
        } 
        else 
        {
            keyPath = "/" + ROOT_PATH_LOCK + "/" + path;
        }
        try 
        {
            final PathChildrenCache cache = new PathChildrenCache(curatorFramework, keyPath, false);
            cache.start(PathChildrenCache.StartMode.POST_INITIALIZED_EVENT);
            cache.getListenable().addListener((client, event) -> {
                if (event.getType().equals(PathChildrenCacheEvent.Type.CHILD_REMOVED)) 
                {
                    String oldPath = event.getData().getPath();
                    log.info("上一个节点 " + oldPath + " 已经被断开");
                    if (oldPath.contains(path)) 
                    {
                        //释放计数器，让当前的请求获取锁
                        countDownLatch.countDown();
                    }
                }
            });
        } 
        catch (Exception e) 
        {
            log.info("监听是否锁失败!{}", e);
        }
    }

    public void init() 
    {
        curatorFramework = curatorFramework.usingNamespace("lock-namespace");
        String path = "/" + ROOT_PATH_LOCK;
        try 
        {
            if (curatorFramework.checkExists().forPath(path) == null) 
            {
                curatorFramework.create()
                        .creatingParentsIfNeeded()
                        .withMode(CreateMode.PERSISTENT)
                        .withACL(ZooDefs.Ids.OPEN_ACL_UNSAFE)
                        .forPath(path);
            }
            addWatcher(ROOT_PATH_LOCK);
            log.info("root path 的 watcher 事件创建成功");
        } 
        catch (Exception e) 
        {
            log.error("connect zookeeper fail，please check the log >> {}", e.getMessage(), e);
        }
    }
}
```

# 参考

https://github.com/liaozihong/SpringBoot-Learning/tree/master/SpringBoot-Zookeeper-Distributed-Lock