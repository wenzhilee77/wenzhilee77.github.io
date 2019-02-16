---
layout: post
title:  "SpringBoot2.0 整合 MybatisPlus 组件"
categories: SpringBoot MybatisPlus HikariCP
tags:  SpringBoot MybatisPlus HikariCP  
author: wenzhilee77
---

* content
{:toc}

## 数据库配置文件

```
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/test?useSSL=false
    username: root
    password: root
    type: com.zaxxer.hikari.HikariDataSource

    hikari:
      minimum-idle: 3
      auto-commit: true
      idle-timeout: 10000
      pool-name: testHikariCP
      max-lifetime: 1800000
      connection-timeout: 30000


mybatis-plus:
  mapper-locations: classpath:/mappers/*Mapper.xml
  #实体扫描，多个package用逗号或者分号分隔
  typeAliasesPackage: com.wenzhilee77.mybatis.demo.model

  global-config:
    #主键类型  0:"数据库ID自增", 1:"用户输入ID", 2:"全局唯一ID (数字类型唯一ID)", 3:"全局唯一ID UUID";
    id-type: 0
    #字段策略 0:"忽略判断", 1:"非 NULL 判断", 2:"非空判断"
    field-strategy: 2
    #驼峰下划线转换
    db-column-underline: true
    #刷新mapper 调试神器
    refresh-mapper: true
    #数据库大写下划线转换
    capital-mode: true
    #逻辑删除配置（下面3个配置）
    logic-delete-value: 1
    logic-not-delete-value: 0
  configuration:
    #配置返回数据库(column下划线命名&&返回java实体是驼峰命名)，自动匹配无需as（没开启这个，SQL需要写as： select user_id as userId）
    map-underscore-to-camel-case: true
    cache-enabled: false
```

## Mybatis Plus 配置类

```
package com.wenzhilee77.mybatis.demo.config;

import com.baomidou.mybatisplus.core.injector.ISqlInjector;
import com.baomidou.mybatisplus.extension.injector.LogicSqlInjector;
import com.baomidou.mybatisplus.extension.plugins.OptimisticLockerInterceptor;
import com.baomidou.mybatisplus.extension.plugins.PaginationInterceptor;
import com.baomidou.mybatisplus.extension.plugins.PerformanceInterceptor;
import org.mybatis.spring.annotation.MapperScan;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;

@Configuration
@MapperScan("com.wenzhilee77.mybatis.demo.mapper")
public class MybatisPlusConfig {

    /**
     * 分页插件
     */
    @Bean
    public PaginationInterceptor paginationInterceptor() {
        return new PaginationInterceptor();
    }

    /**
     * sql注入器  逻辑删除插件
     * @return
     */
    @Bean
    public ISqlInjector iSqlInjector(){
        return new LogicSqlInjector();
    }

    /**
     * sql性能分析插件，输出sql语句及所需时间
     * @return
     */
    @Bean
    @Profile({"dev","test"})// 设置 dev test 环境开启
    public PerformanceInterceptor performanceInterceptor() {
        return new PerformanceInterceptor();
    }
    /**
     * 乐观锁插件
     * @return
     */
    public OptimisticLockerInterceptor optimisticLockerInterceptor(){
        return new OptimisticLockerInterceptor();
    }
}
```

## 实体类

```
package com.wenzhilee77.mybatis.demo.model;

import com.baomidou.mybatisplus.annotation.IdType;
import com.baomidou.mybatisplus.annotation.TableId;
import com.baomidou.mybatisplus.extension.activerecord.Model;

import java.io.Serializable;

public class User extends Model
{
    @TableId(type = IdType.AUTO)
    private Integer userId;

    private String userName;

    private String password;

    private String phone;

    public Integer getUserId()
    {
        return userId;
    }

    public void setUserId(Integer userId)
    {
        this.userId = userId;
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

    public String getPhone()
    {
        return phone;
    }

    public void setPhone(String phone)
    {
        this.phone = phone;
    }

    @Override
    public String toString()
    {
        return "User{" + "userId=" + userId + ", userName='" + userName + '\'' + ", password='" + password + '\'' + ", phone='" + phone + '\'' + '}';
    }
}
```

## 自定义 Mapper 类

```
package com.wenzhilee77.mybatis.demo.mapper;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import com.wenzhilee77.mybatis.demo.model.User;

/**
 * User Mapper interface
 *
 * @author wenzhilee77
 */
public interface UserMapper extends BaseMapper<User>
{

}
```

## Service 接口类

```
package com.wenzhilee77.mybatis.demo.service;

import com.baomidou.mybatisplus.core.conditions.Wrapper;
import com.wenzhilee77.mybatis.demo.model.User;

import java.util.List;

public interface UserService
{
    int insert(User user);

    void deleteUserById(User user);

    void updateUser(User user);

    List<User> selectUsers(Wrapper<User> wrapper);
}
```

## Service 实现类

```
package com.wenzhilee77.mybatis.demo.service.imp;

import com.baomidou.mybatisplus.core.conditions.Wrapper;
import com.wenzhilee77.mybatis.demo.mapper.UserMapper;
import com.wenzhilee77.mybatis.demo.model.User;
import com.wenzhilee77.mybatis.demo.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class UserServiceImpl implements UserService
{
    @Autowired
    private UserMapper userMapper;

    @Override
    public int insert(User user)
    {
        return userMapper.insert(user);
    }

    @Override
    public void deleteUserById(User user)
    {
        userMapper.deleteById(user.getUserId());
    }

    @Override
    public void updateUser(User user)
    {
        userMapper.updateById(user);
    }

    @Override
    public List<User> selectUsers(Wrapper<User> wrapper)
    {
        return userMapper.selectList(wrapper);
    }

}
```

## Controller 实现类

```
package com.wenzhilee77.mybatis.demo.controller;

import com.baomidou.mybatisplus.core.conditions.query.QueryWrapper;
import com.wenzhilee77.mybatis.demo.model.User;
import com.wenzhilee77.mybatis.demo.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/user")
public class UserController
{

    @Autowired
    private UserService userService;

    @PostMapping("")
    public ResponseEntity addUser(@RequestParam(value = "userName", required = true) String userName, @RequestParam(value = "password", required = true) String password, @RequestParam(value = "phone", required = false) String phone)
    {
        User userDomain = new User();
        userDomain.setUserName(userName);
        userDomain.setPassword(password);
        userDomain.setPhone(phone);
        userService.insert(userDomain);
        return ResponseEntity.ok("添加成功");
    }

    @DeleteMapping("")
    public ResponseEntity deleteUser(User user)
    {
        System.out.println(user);
        userService.deleteUserById(user);
        return ResponseEntity.ok("删除成功");
    }

    @PutMapping("")
    public ResponseEntity updateUser(@RequestParam(value = "userId", required = true) Integer userId, @RequestParam(value = "userName", required = false) String userName, @RequestParam(value = "password", required = false) String password, @RequestParam(value = "phone", required = false) String phone)
    {
        User userDomain = new User();
        userDomain.setUserId(userId);
        userDomain.setUserName(userName);
        userDomain.setPassword(password);
        userDomain.setPhone(phone);
        System.out.println(userDomain);
        userService.updateUser(userDomain);
        return ResponseEntity.ok("更新成功");
    }

    @GetMapping("")
    public ResponseEntity getUsers()
    {
        QueryWrapper<User> wrapper = new QueryWrapper<>();
        wrapper.ne("user_id", 3);

        return ResponseEntity.ok(userService.selectUsers(wrapper));
    }
}
```

## 测试启动类

```
package com.wenzhilee77.mybatis.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.transaction.annotation.EnableTransactionManagement;

@SpringBootApplication
@EnableTransactionManagement
@ComponentScan(basePackages = {
        "com.wenzhilee77.mybatis.demo.config",
        "com.wenzhilee77.mybatis.demo.controller",
        "com.wenzhilee77.mybatis.demo.service"})
public class MybatisDemoApplication
{
    public static void main(String[] args)
    {
        SpringApplication.run(MybatisDemoApplication.class, args);
    }
}
```

## 关于分页的简单实现

```
    public void selectPage()
    {
        Page<User> page = new Page<>(1, 5);
        IPage<User> list =  userMapper.selectPage(page, null);

        System.out.println("总记录数：" + page.getTotal());
        System.out.println("总页数" + page.getPages());
        System.out.println(list);
    }
```

参考

https://gitee.com/baomidou/mybatis-plus-samples