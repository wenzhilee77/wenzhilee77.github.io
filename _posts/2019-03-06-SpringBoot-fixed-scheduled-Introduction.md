---
layout: post
title:  "SpringBoot定时任务简介"
categories: schedule
tags: schedule
author: wenzhilee77
---

# 定时器的使用分析

SpingBoot的定时器我相信大家都会使用,即使不会使用的,随便搜索一下,相信都能在30分钟内上手.但是其实Spring的定时器有三种模式,分别是fixedDelay、cron、fixedRate.那这三种分别有什么区别呢?我们来分别写一个SpringBoot的定时器Demo来体验一下.


## fixDelay

```
@Component
 public class ScheduleHandle 
 {
    private final Logger log = LoggerFactory.getLogger(ScheduleHandle.class);
    private List<Integer> index = Arrays.asList(8 * 1000, 3 * 1000, 6 * 1000, 2 * 1000, 2 * 1000);

    private AtomicInteger atomicInteger = new AtomicInteger(0);

    @Scheduled(fixedDelay = 3 * 1000)
    public void fixedDelay() throws Exception 
    {
        int i = atomicInteger.get();
        if (i < 5) 
        {
            Integer sleepTime = index.get(i);
            log.info("第{}个任务开始执行,执行时间为{}ms", i, sleepTime);
            Thread.sleep(sleepTime);
            atomicInteger.getAndIncrement();
        }
    }
}
```

该方式最简单,该方式的意思是,在上一个任务执行完成之后,间隔3秒(因为@Scheduled(fixedDelay = 3 * 1000))后,执行下一个任务.这种是最容易理解的,所以放在第一个来讲.用一个图来表示的话,更容易理解.如下:

![](/images/schedule/schedule1)

可以用输出日志验证上述结论

```
12019-04-07 21:59:11.761  INFO 29372 --- [pool-1-thread-1] com.toby.demo.job.ScheduleHandle         : 第0个任务开始执行,执行时间为8000ms
22019-04-07 21:59:22.772  INFO 29372 --- [pool-1-thread-1] com.toby.demo.job.ScheduleHandle         : 第1个任务开始执行,执行时间为3000ms
32019-04-07 21:59:28.777  INFO 29372 --- [pool-1-thread-1] com.toby.demo.job.ScheduleHandle         : 第2个任务开始执行,执行时间为6000ms
42019-04-07 21:59:37.783  INFO 29372 --- [pool-1-thread-1] com.toby.demo.job.ScheduleHandle         : 第3个任务开始执行,执行时间为2000ms
52019-04-07 21:59:42.785  INFO 29372 --- [pool-1-thread-1] com.toby.demo.job.ScheduleHandle         : 第4个任务开始执行,执行时间为2000ms
```

## corn

```
@Scheduled(cron = "0/5 * * * * ? ")
public void cron() throws Exception 
{
    int i = atomicInteger.get();
    if (i < 5) 
    {
        Integer sleepTime = index.get(i);
        log.info("第{}个任务开始执行,执行时间为{}ms", i, sleepTime);
        Thread.sleep(sleepTime);
        atomicInteger.getAndIncrement();
    }
}
```

因为上方代码配置如下:

```
//每5s执行一次
@Scheduled(cron = "0/5 * * * * ? ")
```

因此你可以理解为5s就是一个周期.这就相当于在宿舍洗澡,因为只有一个洗澡位置(单线程),所以每次只能进去一个人,然后舍长在门口,每5s看一下有没有空位,有空位的话叫下一个进去洗澡.

* 第5秒的时候,舍长看了一下,发现第一个同学还没有出来.

* 第二个周期,也就是第10秒的时候再看一下.发现已经有空位了,那么就叫第二个同学进去洗.

* 第三个周期,也就是15秒的时候,又瞄了一眼,发现有空位了,叫第三个同学进去洗.

* 第四个周期,也就是20秒的时候,发现没有空位.

* 第五个周期的时候,也就是25秒的时候.发现有空位了,接着叫下一个进去洗.剩下的不再分析.

用一张图来表示如下:

![](/images/schedule/schedule2)

可以用输出日志验证上述结论

```
12019-04-07 22:15:30.002  INFO 29385 --- [pool-1-thread-1] com.toby.demo.job.ScheduleHandle         : 第0个任务开始执行,执行时间为8000ms
22019-04-07 22:15:40.001  INFO 29385 --- [pool-1-thread-1] com.toby.demo.job.ScheduleHandle         : 第1个任务开始执行,执行时间为3000ms
32019-04-07 22:15:45.001  INFO 29385 --- [pool-1-thread-1] com.toby.demo.job.ScheduleHandle         : 第2个任务开始执行,执行时间为6000ms
42019-04-07 22:15:55.001  INFO 29385 --- [pool-1-thread-1] com.toby.demo.job.ScheduleHandle         : 第3个任务开始执行,执行时间为2000ms
52019-04-07 22:16:00.001  INFO 29385 --- [pool-1-thread-1] com.toby.demo.job.ScheduleHandle         : 第4个任务开始执行,执行时间为2000ms
```

## fixedRate

```
@Scheduled(fixedRate = 5 * 1000)
public void fixedRate() throws Exception 
{
    int i = atomicInteger.get();
    if (i < 5) 
    {
        Integer sleepTime = index.get(i);
        log.info("第{}个任务开始执行,执行时间为{}ms", i, sleepTime);
        Thread.sleep(sleepTime);
        atomicInteger.getAndIncrement();
    }
}
```

还是要拿洗澡的这个例子来说.但是该方式和cron的方式有很大不同

因上方代码配置如下:

```
@Scheduled(fixedRate = 5 * 1000)
```

你可以理解为舍长预算每个同学洗澡的时间是5秒,但是第一个同学进去洗澡,用了8秒.

* 第二个同学本来应该在第5秒的时候就进去的,结果第一个同学出来的时候,已经是第8秒了,那么舍长就赶紧催第二个同学进去,把时间进度追回来.

* 第二个同学知道时间紧,洗了3秒就出来.此时舍长发现,第三个同学,原本应该是在第10秒进去的,结果现在已经到了第11秒(8+3),那么就赶紧催第三个同学进去.

* 第三个同学沉醉其中,难以自拔的洗了6秒.出来的时候已经是第17秒(8+3+6).舍长掐指一算,发现第四个同学原本应该是第15秒的时候就进去了.结果现在都17秒了,时间不等人,催促第四个同学进去赶紧洗.

* 第四个同学只洗了2秒就出来了,出来的时候,舍长看了一下时间,是第19秒."有原则"的舍长发现,第5个同学原本预算是20秒的时候进去的,结果现在才19秒,不行,那让他在外面玩1秒的手机,等20秒的时候再进去.

用一张图来表示如下:

![](/images/schedule/schedule3)

可以用输出日志验证上述结论

```
12019-04-07 22:18:44.814  INFO 29390 --- [pool-1-thread-1] com.toby.demo.job.ScheduleHandle         : 第0个任务开始执行,执行时间为8000ms
22019-04-07 22:18:52.819  INFO 29390 --- [pool-1-thread-1] com.toby.demo.job.ScheduleHandle         : 第1个任务开始执行,执行时间为3000ms
32019-04-07 22:18:55.824  INFO 29390 --- [pool-1-thread-1] com.toby.demo.job.ScheduleHandle         : 第2个任务开始执行,执行时间为6000ms
42019-04-07 22:19:01.829  INFO 29390 --- [pool-1-thread-1] com.toby.demo.job.ScheduleHandle         : 第3个任务开始执行,执行时间为2000ms
52019-04-07 22:19:04.816  INFO 29390 --- [pool-1-thread-1] com.toby.demo.job.ScheduleHandle         : 第4个任务开始执行,执行时间为2000ms
```

参考链接：

原理暂且不谈，定时器你当真会用？ - 肥朝

