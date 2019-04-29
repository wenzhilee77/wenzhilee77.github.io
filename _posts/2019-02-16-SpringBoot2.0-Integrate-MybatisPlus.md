---
layout: post
title:  "SpringBoot2.0整合MybatisPlus"
categories: SpringBoot Mybatis-Plus HikariCP
tags:  SpringBoot Mybatis-Plus HikariCP  
author: wenzhilee77
---

* content
{:toc}

# 依赖jar包

pom.xml
```
<dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.15</version>
        </dependency>

        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>3.1.1</version>
        </dependency>
    </dependencies>
```


# 数据库配置文件

application.xml
```
server:
  port: 8998

spring:
  datasource:
    url: jdbc:mysql://10.179.69.34:3306/online?useSSL=false&useUnicode=true&characterEncoding=UTF-8
    username: root
    password:
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
  typeAliasesPackage: com.wenzhli.integrate.data.model

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


# Mybatis Plus 配置类

```
@Configuration
@MapperScan("com.wenzhli.integrate.data.mapper")
public class MybatisPlusConfig
{

    /**
     * 分页插件
     */
    @Bean
    public PaginationInterceptor paginationInterceptor()
    {
        return new PaginationInterceptor();
    }

    /**
     * sql注入器  逻辑删除插件
     *
     * @return
     */
    @Bean
    public ISqlInjector iSqlInjector()
    {
        return new LogicSqlInjector();
    }

    /**
     * sql性能分析插件，输出sql语句及所需时间
     *
     * @return
     */
    @Bean
    @Profile({"dev", "test"})// 设置 dev test 环境开启
    public PerformanceInterceptor performanceInterceptor()
    {
        return new PerformanceInterceptor();
    }

    /**
     * 乐观锁插件
     *
     * @return
     */
    public OptimisticLockerInterceptor optimisticLockerInterceptor()
    {
        return new OptimisticLockerInterceptor();
    }
}
```


# 实体类

```
public class UserInfo extends Model
{
    @TableId
    private int id;
    private String usernameZh;
    private String phone;
    private String idCard;
    private String email;
    private String organization;
    private String department;
    private String job;
    private String city;
    private int status;
    private Date createTime;
    private Date updateTime;
    private int creatorId;
    private int isDelete;

    public int getId()
    {
        return id;
    }

    public void setId(int id)
    {
        this.id = id;
    }

    public String getUsernameZh()
    {
        return usernameZh;
    }

    public void setUsernameZh(String usernameZh)
    {
        this.usernameZh = usernameZh;
    }

    public String getPhone()
    {
        return phone;
    }

    public void setPhone(String phone)
    {
        this.phone = phone;
    }

    public String getIdCard()
    {
        return idCard;
    }

    public void setIdCard(String idCard)
    {
        this.idCard = idCard;
    }

    public String getEmail()
    {
        return email;
    }

    public void setEmail(String email)
    {
        this.email = email;
    }

    public String getOrganization()
    {
        return organization;
    }

    public void setOrganization(String organization)
    {
        this.organization = organization;
    }

    public String getDepartment()
    {
        return department;
    }

    public void setDepartment(String department)
    {
        this.department = department;
    }

    public String getJob()
    {
        return job;
    }

    public void setJob(String job)
    {
        this.job = job;
    }

    public String getCity()
    {
        return city;
    }

    public void setCity(String city)
    {
        this.city = city;
    }

    public int getStatus()
    {
        return status;
    }

    public void setStatus(int status)
    {
        this.status = status;
    }

    public Date getCreateTime()
    {
        return createTime;
    }

    public void setCreateTime(Date createTime)
    {
        this.createTime = createTime;
    }

    public Date getUpdateTime()
    {
        return updateTime;
    }

    public void setUpdateTime(Date updateTime)
    {
        this.updateTime = updateTime;
    }

    public int getCreatorId()
    {
        return creatorId;
    }

    public void setCreatorId(int creatorId)
    {
        this.creatorId = creatorId;
    }

    public int getIsDelete()
    {
        return isDelete;
    }

    public void setIsDelete(int isDelete)
    {
        this.isDelete = isDelete;
    }

    @Override
    public String toString()
    {
        return "UserInfo{" + "id=" + id + ", usernameZh='" + usernameZh + '\'' + ", phone='" + phone + '\'' + ", idCard='" + idCard + '\'' + ", email='" + email + '\'' + ", organization='" + organization + '\'' + ", department='" + department + '\'' + ", job='" + job + '\'' + ", city='" + city + '\'' + ", status=" + status + ", createTime=" + createTime + ", updateTime=" + updateTime + ", creatorId=" + creatorId + ", isDelete=" + isDelete + '}';
    }
}
```


# 自定义 Mapper 类

```
public interface UserInfoMapper extends BaseMapper<UserInfo>
{
}
```


# Service 接口类

```
public interface UserService
{
    int insert(User user);

    void deleteUserById(User user);

    void updateUser(User user);

    List<User> selectUsers(Wrapper<User> wrapper);
}
```


# Service 实现类

```
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


# Controller 实现类

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


# 测试启动类

```
@SpringBootApplication
@EnableTransactionManagement
@ComponentScan(basePackages = {
        "com.wenzhli.integrate.data.config",
        "com.wenzhli.integrate.data.mapper",
        "com.wenzhli.integrate.data.controller"})
public class DataApplication
{
    public static void main(String[] args)
    {
        SpringApplication.run(DataApplication.class, args);
    }
}
```


# 关于分页的简单实现

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


# 参考

https://gitee.com/baomidou/mybatis-plus-samples