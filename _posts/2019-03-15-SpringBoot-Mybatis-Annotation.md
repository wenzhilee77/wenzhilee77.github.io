# 依赖jar包
```xml
    <properties>
        <java.version>1.8</java.version>
        <mybatis.version>2.0.1</mybatis.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.mybatis.spring.boot</groupId>
                <artifactId>mybatis-spring-boot-starter</artifactId>
                <version>${mybatis.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
```

# 配置文件
```yaml
server:
  port: 8070

spring:
  datasource:
    url: jdbc:mysql://10.179.23.229/traffis_sso?serverTimezone=Asia/Shanghai&useUnicode=true&characterEncoding=UTF-8&useSSL=false
    username: root
    password:
    driver-class-name: com.mysql.cj.jdbc.Driver
```


# 启动程序
```java
@SpringBootApplication
@MapperScan("com.example.demo.mapper")
public class DemoApplication
{

    public static void main(String[] args)
    {
        SpringApplication.run(DemoApplication.class, args);
    }

}
```

# 实体类
```java
public class User
{
    private Long id;
    private String usernameZh;
    private String phone;
    private String idCard;
    private String email;
    private String organization;
    private String department;
    private String job;
    private String city;
    private String status;
    private Date createTime;
    private Date updateTime;
    private Long creatorId;
    private Integer isDelete;

    public Long getId()
    {
        return id;
    }

    public void setId(Long id)
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

    public String getStatus()
    {
        return status;
    }

    public void setStatus(String status)
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

    public Long getCreatorId()
    {
        return creatorId;
    }

    public void setCreatorId(Long creatorId)
    {
        this.creatorId = creatorId;
    }

    public Integer getIsDelete()
    {
        return isDelete;
    }

    public void setIsDelete(Integer isDelete)
    {
        this.isDelete = isDelete;
    }

    @Override
    public String toString()
    {
        return "User{" + "id=" + id + ", usernameZh='" + usernameZh + '\'' + ", phone='" + phone + '\'' + ", idCard='" + idCard + '\'' + ", email='" + email + '\'' + ", organization='" + organization + '\'' + ", department='" + department + '\'' + ", job='" + job + '\'' + ", city='" + city + '\'' + ", status='" + status + '\'' + ", createTime=" + createTime + ", updateTime=" + updateTime + ", creatorId=" + creatorId + ", isDelete=" + isDelete + '}';
    }
}
```

# 测试类
```java
@RequestMapping("/test")
@RestController
public class Test
{
    @Autowired
    private UserMapper userMapper;

    @GetMapping("/getAll")
    List<User> getAll()
    {
        return userMapper.findAllUsers();
    }

    @GetMapping("/insert")
    public void insertUser()
    {
        User user = new User();
        user.setUpdateTime(new Date());
        user.setCreateTime(new Date());
        user.setEmail("test@qq.com");
        user.setIdCard("370380390");
        user.setPhone("11011011011");
        user.setUsernameZh("tdev");

        userMapper.insertUser(user);
    }

    @GetMapping("/delete")
    public void deleteUser(Long id)
    {
        userMapper.deleteUser(id);
    }

    @GetMapping("/updateUser")
    public void updateUser()
    {
        User user = new User();
        user.setId(12922L);
        user.setUpdateTime(new Date());
        user.setCreateTime(new Date());
        user.setEmail("test@qq.com");
        user.setIdCard("370380390");
        user.setPhone("11011011011");
        user.setUsernameZh("test");

        userMapper.updateUser(user);
    }

}
```

# Mapper接口
```java
public interface UserMapper
{
    @Results({
            @Result(property = "usernameZh", column = "username_zh"),
            @Result(property = "idCard", column = "id_card"),
            @Result(property = "createTime", column = "create_time"),
            @Result(property = "updateTime", column = "update_time"),
            @Result(property = "isDelete", column = "is_delete")

    })
    @Select("select * from user_info")
    List<User> findAllUsers();

    @Insert("insert into user_info(username_zh, phone, id_card, email, create_time, update_time) " +
            "values(#{usernameZh}, #{phone}, #{idCard}, #{email}, #{createTime}, #{updateTime})")
    void insertUser(User user);

    @Delete("delete from user_info where id = #{id}")
    void deleteUser(Long id);

    @Update("update user_info set username_zh = #{usernameZh} where id = #{id}")
    void updateUser(User user);
}
```