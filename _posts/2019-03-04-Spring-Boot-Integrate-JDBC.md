---
layout: post
title:  "SpringBoot 整合 JDBC"
categories: SpringBoot JDBC
tags:  SpringBoot JDBC
author: wenzhilee77
---

# application.yml

```
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/test
    username: root
    driver-class-name: com.mysql.cj.jdbc.Driver
    password: root
```


# Swagger 配置文件

```
ckage com.wenzhli.jdbc.demo.config;

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
                .apis(RequestHandlerSelectors.basePackage("com.wenzhli.jdbc.demo.web"))
                .paths(PathSelectors.any())
                .build();
    }

}
```

# 函数统一返回类型定义

```
package com.wenzhli.jdbc.demo.model.result;

import java.io.Serializable;

public class ResultBean<T> implements Serializable
{
    public static final int SUCCESS = 200;
    public static final int FAIL = 1;
    public static final int NO_PERMISSION = 2;

    private int code;
    private String msg;
    private T data;

    {
        code = SUCCESS;
        msg = "success";
    }

    public ResultBean()
    {
        super();
    }

    public ResultBean(T data)
    {
        this.data = data;
    }

    public ResultBean(Throwable e)
    {
        super();
        this.code = FAIL;
        this.msg = e.getMessage();
    }

    public int getCode()
    {
        return code;
    }

    public void setCode(int code)
    {
        this.code = code;
    }

    public String getMsg()
    {
        return msg;
    }

    public void setMsg(String msg)
    {
        this.msg = msg;
    }

    public T getData()
    {
        return data;
    }

    public void setData(T data)
    {
        this.data = data;
    }
}
```

# 实体类 User 定义

```
package com.wenzhli.jdbc.demo.model;

public class User
{
    private int userId;
    private String userName;
    private String password;
    private String phone;

    public int getUserId()
    {
        return userId;
    }

    public void setUserId(int userId)
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

# service 接口类的定义

```
package com.wenzhli.jdbc.demo.service;

import com.wenzhli.jdbc.demo.model.User;

public interface UserService
{
    int insertUser(User user);
}
```

# service 接口实现类

```
package com.wenzhli.jdbc.demo.service.impl;

import com.wenzhli.jdbc.demo.model.User;
import com.wenzhli.jdbc.demo.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;

@Service
public class SimpleUserService implements UserService
{
    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Override
    public int insertUser(User user)
    {
        return jdbcTemplate.update("insert into user(user_name, password, phone) values(?, ?, ?)",
                user.getUserName(),
                user.getPassword(),
                user.getPhone());
    }
}
```

# 控制类的定义

```
package com.wenzhli.jdbc.demo.web;

import com.wenzhli.jdbc.demo.model.User;
import com.wenzhli.jdbc.demo.model.result.ResultBean;
import com.wenzhli.jdbc.demo.service.UserService;
import io.swagger.annotations.Api;
import io.swagger.annotations.ApiImplicitParam;
import io.swagger.annotations.ApiOperation;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

@RestController
@Api("test swagger")
@RequestMapping("/users")
public class UserController
{
    @Autowired
    private UserService userService;

    @ApiOperation("insert the user into the db using the post method")
    @ApiImplicitParam(name="User", value="User", dataType = "User", paramType = "path")
    @RequestMapping(value = {"", "/"}, method = RequestMethod.POST)
    public ResultBean insertUser(User u)
    {
        ResultBean result = null;

        try
        {
            int ret = userService.insertUser(u);
            result = new ResultBean();
        }
        catch(Exception e)
        {
            result = new ResultBean(e);
        }
        return result;
    }
}
```

# 测试方法

http://localhost:8080/swagger-ui.html#/