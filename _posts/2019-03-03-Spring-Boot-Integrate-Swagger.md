---
layout: post
title:  "SpringBoot 整合 Swagger"
categories: SpringBoot Swagger
tags:  SpringBoot Swagger
author: wenzhilee77
---

# 相关 jars

```java
<dependencies>
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
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger2</artifactId>
            <version>2.9.2</version>
        </dependency>

        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger-ui</artifactId>
            <version>2.9.2</version>
        </dependency>
    </dependencies>
```

# Swagger2 配置文件

```java
package com.wenzhli.swagger.demo.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import springfox.documentation.builders.PathSelectors;
import springfox.documentation.builders.RequestHandlerSelectors;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spring.web.plugins.Docket;
import springfox.documentation.swagger2.annotations.EnableSwagger2;

@Configuration
@EnableSwagger2
public class SwaggerConfig
{
    @Bean
    public Docket createRestApi()
    {
        return new Docket(DocumentationType.SWAGGER_2)
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.wenzhli.swagger.demo.web"))
                .paths(PathSelectors.any())
                .build();
    }

}
```

# model类

```java
package com.wenzhli.swagger.demo.model;

public class User
{
    private long id;
    private String name;
    private int age;

    public User()
    {

    }

    public User(long id, String name, int age)
    {
        this.id = id;
        this.name = name;
        this.age = age;
    }

    public long getId()
    {
        return id;
    }

    public void setId(long id)
    {
        this.id = id;
    }

    public String getName()
    {
        return name;
    }

    public void setName(String name)
    {
        this.name = name;
    }

    public int getAge()
    {
        return age;
    }

    public void setAge(int age)
    {
        this.age = age;
    }

    @Override
    public String toString()
    {
        return "User{" + "id=" + id + ", name='" + name + '\'' + ", age=" + age + '}';
    }
}
```

# 控制类

```java
package com.wenzhli.swagger.demo.web;

import com.wenzhli.swagger.demo.model.User;
import io.swagger.annotations.*;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/users")
@Api(value="cruds for users")
public class UserController
{
    private static Map<Long, User> users = new HashMap<>();

    static
    {
        users.put(1L, new User(1, "test1", 10));
        users.put(2L, new User(2, "test2", 20));
        users.put(3L, new User(3, "test3", 30));
    }


    @ApiOperation("get all the users in db")
    @RequestMapping(value = {"", "/"}, method = RequestMethod.GET)
    public List<User> getAllUsers()
    {
        return new ArrayList<>(users.values());
    }

    @ApiOperation(value = "Get user by id")
    @ApiImplicitParam(name = "id", value = "this is the user id", required = true, paramType = "path")
    @RequestMapping(value="/{id}", method = RequestMethod.GET)
    public User getUserById(/*@ApiParam(value = "this is user id", required = true)*/ @PathVariable long id)
    {
        return users.get(id);
    }

    @ApiOperation("update the user using the put method")
    @ApiImplicitParams({
            @ApiImplicitParam(name = "id", value = "this is the user id", required = true, dataType = "Long", paramType = "path"),
            @ApiImplicitParam(name = "user", value = "User", required = true, paramType = "User")})
    @RequestMapping(value={"/{id}"}, method = RequestMethod.PUT)
    public void putUser(@PathVariable long id, User user)
    {
        User u = users.get(id);
        u.setAge(user.getAge());
        u.setName(user.getName());
        users.put(id, u);
    }

    @ApiOperation("delete user by id")
    @ApiImplicitParam(name = "id", value = "this is the user id", required = true, dataType = "Long", paramType = "path")
    @RequestMapping(value = "/{id}", method = RequestMethod.DELETE)
    public void deleteUserById(@PathVariable long id)
    {
        users.remove(id);
    }
}
```

# 测试方法

http://localhost:8080/swagger-ui.html#/