---
layout: post
title:  "SpringBoot 整合 MockMvc"
categories: SpringBoot MockMvc
tags:  SpringBoot MockMvc
author: wenzhilee77
---

# 引入 spring-boot-starter-test

```java
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
    </dependency>
</dependencies>
```

# 测试实体类

```java
public class User
{
    private String name;
    private int age;
    private long id;

    public User()
    {

    }

    public User(String name, int age, long id)
    {
        this.name = name;
        this.age = age;
        this.id = id;
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

    @Override
    public String toString()
    {
        return "User{" + "name='" + name + '\'' + ", age=" + age + ", id=" + id + '}';
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
}
```

# UserController类

```java
import com.wenzhli.restful.demo.domain.User;
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
public class UserController
{
    private static Map<Long, User> users = new HashMap<>();

    static
    {
        users.put(1L, new User("test", 10, 1));
        users.put(2L, new User("test2", 20, 2));
    }

    @RequestMapping(value="/", method = RequestMethod.GET)
    public List<User> getUserList()
    {
        return new ArrayList<>(users.values());
    }

    @RequestMapping(value="/{id}", method = RequestMethod.GET)
    public User getUserById(@PathVariable long id)
    {
        return users.get(id);
    }

    @RequestMapping(value="/", method=RequestMethod.POST)
    public String postUser(User user)
    {
        users.put(user.getId(), user);
        return "success";
    }

    @RequestMapping(value="/", method = RequestMethod.PUT)
    public String putUser(User user)
    {
        User u = users.get(user.getId());
        u.setAge(user.getAge());
        u.setName(user.getName());
        return "success";
    }

    @RequestMapping(value="/{id}", method =RequestMethod.DELETE)
    public String deleteUser(@PathVariable Long id)
    {
        users.remove(id);
        return "success";
    }
}
```

# UserControllerTest类

```java
import com.wenzhli.restful.demo.DemoApplication;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.RequestBuilder;

import static org.hamcrest.CoreMatchers.equalTo;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.print;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@RunWith(SpringRunner.class)
@SpringBootTest(classes = DemoApplication.class)
@AutoConfigureMockMvc
public class UserControllerTest
{
    @Autowired
    private MockMvc mockMvc;

    @Test
    public void testGetUserList() throws Exception
    {
        mockMvc.perform(get("/users/"))
                .andDo(print())
                .andExpect(content().string(equalTo("[{\"name\":\"test\",\"age\":10,\"id\":1},{\"name\":\"test2\",\"age\":20,\"id\":2}]")))
                .andExpect(status().isOk());
    }


    @Test
    public void testGetUserById() throws Exception
    {
        mockMvc.perform(get("/users/{id}", 1))
                .andDo(print())
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.name").value("test"))
                .andExpect(jsonPath("$.age").value(10))
                .andExpect(jsonPath("$.id").value(1));
    }


    @Test
    public void testPostUser() throws Exception
    {
        mockMvc.perform(post("/users/")
                .param("name", "test3")
                .param("age", "30")
                .param("id", "3")
        .contentType(MediaType.APPLICATION_JSON)
        .accept(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo("success")));
    }


    @Test
    public void testPutUser() throws Exception
    {
        mockMvc.perform(put("/users/")
                .param("name", "ruhua")
                .param("age", "18")
                .param("id", "1")
        .contentType(MediaType.APPLICATION_JSON)
        .accept(MediaType.APPLICATION_JSON))
                .andExpect(content().string(equalTo("success")))
                .andExpect(status().isOk());
    }


    @Test
    public void testDeleteUser() throws Exception
    {
        mockMvc.perform(delete("/users/{id}", 1)
        .accept(MediaType.APPLICATION_JSON)
        .contentType(MediaType.APPLICATION_JSON)
        .param("id", "1"))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo("success")));
    }


    @Test
    public void testAllFuncs() throws Exception
    {
        RequestBuilder request = null;

        // get all the users in db
        request = get("/users/");
        mockMvc.perform(request)
                .andDo(print())
                .andExpect(status().isOk());

        // get the user by id
        request = get("/users/{id}", 1);
        mockMvc.perform(request)
                .andExpect(content().string(equalTo("{\"name\":\"test\",\"age\":10,\"id\":1}")))
                .andDo(print())
                .andExpect(status().isOk());

        // add a user by the post method
        request = post("/users/")
                .contentType(MediaType.APPLICATION_JSON)
                .accept(MediaType.APPLICATION_JSON)
                .param("name", "test3")
                .param("age", "30")
                .param("id", "3");
        mockMvc.perform(request)
                .andDo(print())
                .andExpect(content().string(equalTo("success")))
                .andExpect(status().isOk());

        // update the person by the put method
        request = put("/users/")
                .param("name", "ruhua")
                .param("age", "40")
                .param("id", "1")
                .contentType(MediaType.APPLICATION_JSON)
                .accept(MediaType.APPLICATION_JSON);
        mockMvc.perform(request)
                .andDo(print())
                .andExpect(content().string(equalTo("success")))
                .andExpect(status().isOk());

        // delete the person by the delete method
        request = delete("/users/{id}", 2);
        mockMvc.perform(request)
                .andDo(print())
                .andExpect(content().string(equalTo("success")))
                .andExpect(status().isOk());

        // show all the users by the end
        request = get("/users/");
        mockMvc.perform(request)
                .andDo(print())
                .andExpect(content().string(equalTo("[{\"name\":\"ruhua\",\"age\":40,\"id\":1},{\"name\":\"test3\",\"age\":30,\"id\":3}]")));
    }
} 
```


