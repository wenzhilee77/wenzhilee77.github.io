---
layout: post
title:  SpringBoot使用Interceptor
categories: SpringBoot Interceptor
tags: SpringBoot Interceptor
author: wenzhilee77
---

# config
```java
@Configuration
public class MVCIntercepterConfig implements WebMvcConfigurer
{
    @Autowired
    private Intercepter1 intercepter1;

    @Autowired
    private Intercepter2 intercepter2;

    @Autowired
    private Intercepter3 intercepter3;

    @Override
    public void addInterceptors(InterceptorRegistry registry)
    {
        registry.addInterceptor(intercepter3).addPathPatterns("/**");
        registry.addInterceptor(intercepter2).addPathPatterns("/**");
        registry.addInterceptor(intercepter1).addPathPatterns("/**");
    }
}
```

# controller
```java
@RequestMapping("/test")
@RestController
public class Test
{
    @GetMapping("/hello")
    public String hello()
    {
    	System.out.println("Test interceptor...");
        return "hello";
    }
}
```

# interceptors
```java
@Component
public class Intercepter1 implements HandlerInterceptor
{
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception
    {
        System.out.println("Intercepter 1-preHandle");
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception
    {
        System.out.println("Intercepter 1-postHandle");
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception
    {
        System.out.println("Intercepter 1-afterCompletion");
    }
}

@Component
public class Intercepter2 implements HandlerInterceptor
{
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception
    {
        System.out.println("Intercepter 2-preHandle");
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception
    {
        System.out.println("Intercepter 2-postHandle");
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception
    {
        System.out.println("Intercepter 2-afterCompletion");
    }
}

@Component
public class Intercepter3 implements HandlerInterceptor
{
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception
    {
        System.out.println("Intercepter 3-preHandle");
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception
    {
        System.out.println("Intercepter 3-postHandle");
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception
    {
        System.out.println("Intercepter 3-afterCompletion");
    }
}
```

# Results
```java
Intercepter 3-preHandle
Intercepter 2-preHandle
Intercepter 1-preHandle
Test interceptor...
Intercepter 1-postHandle
Intercepter 2-postHandle
Intercepter 3-postHandle
Intercepter 1-afterCompletion
Intercepter 2-afterCompletion
Intercepter 3-afterCompletion
```