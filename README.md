# 权限测试项目

### 添加Spring Security的依赖
```
<dependency>
            <groupId>org.thymeleaf.extras</groupId>
            <artifactId>thymeleaf-extras-springsecurity4</artifactId>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.40</version>
        </dependency>
```

### 配置application.properties

```

spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/sang?useUnicode=true&characterEncoding=utf-8
spring.datasource.username=root
spring.datasource.password=sang
logging.level.org.springframework.security=info
spring.thymeleaf.cache=false
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

### 定义用户和角色
```

@Entity
public class SysRole {
    @Id
    @GeneratedValue
    private Long id;
    private String name;


    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

### 定义用户

```
@Entity
public class SysUser implements UserDetails {
    @Id
    @GeneratedValue
    private Long id;
    private String username;
    private String password;

    @ManyToMany(cascade = {CascadeType.REFRESH},fetch = FetchType.EAGER)
    private List<SysRole> roles;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public List<SysRole> getRoles() {
        return roles;
    }

    public void setRoles(List<SysRole> roles) {
        this.roles = roles;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        List<GrantedAuthority> auths = new ArrayList<>();
        List<SysRole> roles = this.getRoles();
        for (SysRole role : roles) {
            auths.add(new SimpleGrantedAuthority(role.getName()));
        }
        return auths;
    }

    @Override
    public String getPassword() {
        return this.password;
    }

    @Override
    public String getUsername() {
        return this.username;
    }

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return true;
    }
}
```

### 预设数据

```

insert  into `sys_role`(`id`,`name`) values (1,'ROLE_ADMIN'),(2,'ROLE_USER');

insert  into `sys_user`(`id`,`password`,`username`) values (1,'root','root'),(2,'sang','sang');

insert  into `sys_user_roles`(`sys_user_id`,`roles_id`) values (1,1),(2,2);
```



### 创建传值对象
```
public class Msg {
    private String title;
    private String content;
    private String extraInfo;

    public Msg() {
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }

    public String getExtraInfo() {
        return extraInfo;
    }

    public void setExtraInfo(String extraInfo) {
        this.extraInfo = extraInfo;
    }

    public Msg(String title, String content, String extraInfo) {
        this.title = title;
        this.content = content;
        this.extraInfo = extraInfo;
    }
}
```



### 创建数据访问接口
```
public interface SysUserRepository extends JpaRepository<SysUser, Long> {
    SysUser findByUsername(String username);
}
```

### 自定义UserDetailsService，实现相应的接口
```
public class CustomUserService implements UserDetailsService {
    @Autowired
    SysUserRepository userRepository;
    @Override
    public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {
        SysUser user = userRepository.findByUsername(s);
        if (user == null) {
            throw new UsernameNotFoundException("用户名不存在");
        }
        System.out.println("s:"+s);
        System.out.println("username:"+user.getUsername()+";password:"+user.getPassword());
        return user;
    }
}
```
### SpringMVC配置
```
@Configuration
public class WebMvcConfig extends WebMvcConfigurerAdapter {
    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/login").setViewName("login");
    }
}
```

### 配置Spring Security
```
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Bean
    UserDetailsService customUserService() {
        return new CustomUserService();
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(customUserService());
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .anyRequest().authenticated()
                .and().formLogin().loginPage("/login").failureUrl("/login?error").permitAll().and()
                .logout().permitAll();
    }
}
```
### 创建登录页面
- 在template文件夹中创建login.html页面
```
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8"/>
    <title>登录</title>
    <link rel="stylesheet" th:href="@{css/bootstrap.min.css}"/>
    <link rel="stylesheet" th:href="@{css/signin.css}"/>
    <style type="text/css">
        body {
            padding-top: 50px;
        }

        .starter-template {
            padding: 40px 15px;
            text-align: center;
        }
    </style>
</head>
<body>
<nav class="navbar navbar-inverse navbar-fixed-top">
    <div class="container">
        <div class="navbar-header">
            <a class="navbar-brand" href="#">Spring Security演示</a>
        </div>
        <div id="navbar" class="collapse navbar-collapse">
            <ul class="nav navbar-nav">
                <li><a th:href="@{/}">首页</a></li>
                <li><a th:href="@{http://www.baidu.com}">百度</a></li>
            </ul>
        </div>
    </div>
</nav>
<div class="container">
    <div class="starter-template">
        <p th:if="${param.logout}" class="bg-warning">已注销</p>
        <p th:if="${param.error}" class="bg-danger">有错误，请重试</p>
        <h2>使用账号密码登录</h2>
        <form class="form-signin" role="form" name="form" th:action="@{/login}" action="/login" method="post">
            <div class="form-group">
                <label for="username">账号</label>
                <input type="text" class="form-control" name="username" value="" placeholder="账号"/>
            </div>
            <div class="form-group">
                <label for="password">密码</label>
                <input type="password" class="form-control" name="password" placeholder="密码"/>
            </div>
            <input type="submit" id="login" value="Login" class="btn btn-primary"/>
        </form>
    </div>
</div>
</body>
</html>
```
### 创建登录成功后跳转页面
```
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org"
      xmlns:sec="http://www.thymeleaf.org/thymeleaf-extras-springsecurity4">
<head>
    <meta charset="UTF-8"/>
    <title sec:authentication="name"></title>
    <link rel="stylesheet" th:href="@{css/bootstrap.min.css}"/>
    <style type="text/css">
        body {
            padding-top: 50px;
        }

        .starter-template {
            padding: 40px 15px;
            text-align: center;
        }
    </style>
</head>
<body>
<nav class="navbar navbar-inverse navbar-fixed-top">
    <div class="container">
        <div class="navbar-header">
            <a class="navbar-brand" href="#">Spring Security演示</a>
        </div>
        <div id="navbar" class="collapse navbar-collapse">
            <ul class="nav navbar-nav">
                <li><a th:href="@{/}">首页</a></li>
                <li><a th:href="@{http://www.baidu.com}">百度</a></li>
            </ul>
        </div>
    </div>
</nav>
<div class="container">
    <div class="starter-template">
        <h1 th:text="${msg.title}"></h1>
        <p class="bg-primary" th:text="${msg.content}"></p>
        <div sec:authorize="hasRole('ROLE_ADMIN')">
            <p class="bg-info" th:text="${msg.extraInfo}"></p>
        </div>
        <div sec:authorize="hasRole('ROLE_USER')">
            <p class="bg-info">无更多显示信息</p>
        </div>
        <form th:action="@{/logout}" method="post">
            <input type="submit" class="btn btn-primary" value="注销"/>
        </form>
    </div>
</div>
</body>
</html>
```

### 添加控制器
```
@Controller
public class HomeController {
    @RequestMapping("/")
    public String index(Model model) {
        Msg msg = new Msg("测试标题", "测试内容", "额外信息，只对管理员显示");
        model.addAttribute("msg", msg);
        return "index";
    }
}
```

### 首页
```
http://localhost:8080/
```