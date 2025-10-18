# Spring

## Spring 常用注解

### 1. @SpringBootApplication

这里先单独拎出`@SpringBootApplication` 注解说一下，虽然我们一般不会主动去使用它。

*这个注解是 Spring Boot 项目的基石，创建 SpringBoot 项目之后会默认在主类加上。*

```java
@SpringBootApplication
public class SpringSecurityJwtGuideApplication {
      public static void main(java.lang.String[] args) {
        SpringApplication.run(SpringSecurityJwtGuideApplication.class, args);
    }
}
```

我们可以把 `@SpringBootApplication`看作是 `@Configuration`、`@EnableAutoConfiguration`、`@ComponentScan` 注解的集合。

```java
package org.springframework.boot.autoconfigure;
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
    @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
    @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
   ......
}

package org.springframework.boot;
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
public @interface SpringBootConfiguration {

}
```

根据 SpringBoot 官网，这三个注解的作用分别是：

- `@EnableAutoConfiguration`：启用 SpringBoot 的自动配置机制
- `@ComponentScan`：扫描被`@Component` (`@Repository`,`@Service`,`@Controller`)注解的 bean，注解默认会扫描该类所在的包下所有的类。
- `@Configuration`：允许在 Spring 上下文中注册额外的 bean 或导入其他配置类



### 2. Spring Bean 相关

#### 2.1. @Autowired与@Resource

**@Autowired**

自动导入对象到类中，被注入进的类同样要被 Spring 容器管理比如：**Service 类注入到 Controller 类中**。

```java
@Service
public class UserService {
  ......
}

@RestController
@RequestMapping("/users")
public class UserController {
   @Autowired
   private UserService userService;
   ......
}
```

@Autowired 和 @Resource 的区别主要体现在以下 5 点：

1. 来源不同；
2. 依赖查找的顺序不同；
3. 支持的参数不同；
4. 依赖注入的用法不同；
5. 编译器 IDEA 的提示不同。

==1）**来源不同**==

**@Autowired 和 @Resource 来自不同的“父类”，其中 @Autowired 是 Spring 定义的注解，而 @Resource 是 Java 定义的注解，它来自于 JSR-250（Java 250 规范提案）。**

==2）**依赖查找顺序不同**==

依赖注入的功能，是通过先在 Spring IoC 容器中查找对象，再将对象注入引入到当前类中。而查找有分为两种实现：**按名称（byName）查找或按类型（byType）查找**，其中 @Autowired 和 @Resource 都是既使用了名称查找又使用了类型查找，但二者进行查找的顺序却截然相反。

-  **@Autowired 查找顺序**
   - **@Autowired 先根据类型（byType）查找，如果存在多个（Bean）再根据名称（byName）进行查找；**

@Autowired默认按类型装配（这个注解是属于spring的），**默认情况下必须要求依赖对象必须存在，如果要允许null值，可以设置它的required属性为false，如：@Autowired(required=false)** ，如果我们想使用名称装配可以结合@Qualifier注解进行使用，如下：

```java
@Autowired
@Qualifier("baseDao")
privateBaseDao baseDao;
```

- **@Qualifier注解的作用**

​	@Autowired是根据类型进行自动装配的。如果当Spring上下文中存在不止一个UserDao类型的bean时，就会抛出BeanCreationException异常;如果Spring上下文中不存在UserDao类型的bean，也会抛出BeanCreationException异常。我们可以使用@Qualifier配合@Autowired来解决这些问题

**①可能存在多个UserDao实例**

```java
@Autowired 
@Qualifier("userServiceImpl") 
public IUserService userService; 

或者

@Autowired
public void setUserDao(@Qualifier("userDao") UserDao userDao) {
　　this.userDao = userDao;
}
```

这样Spring会找到id为userServiceImpl和userDao的bean进行装配。

**②可能不存在UserDao实例**

```java
@Autowired(required = false) 
public IUserService userService
```

- **@Resource 查找顺序**
  - **@Resource 先根据名称（byName）查找，如果（根据名称）查找不到，再根据类型（byType）进行查找。**

​	@Resource（这个注解属于J2EE的），默认按照名称进行装配，名称可以通过name属性进行指定，**如果没有指定name属性，当注解写在字段上时，默认取字段名进行安装名称查找，如果注解写在setter方法上默认取属性名进行装配**。当找不到与名称匹配的bean时才按照类型进行装配。但是需要注意的是，如果name属性一旦指定，就只会按照名称进行装配。

- 推荐使用：@Resource注解在字段上，这样就不用写setter方法了，并且这个注解是属于J2EE的，减少了与spring的耦合。这样代码看起就比较优雅。

==3）**支持的参数不同**==

@Autowired 和 @Resource 在使用时都可以设置参数，比如给 @Resource 注解设置 name 和 type 参数，实现代码如下：

```java
@Resource(name = "userinfo", type = UserInfo.class)
private UserInfo user;
```

但**二者支持的参数以及参数的个数完全不同，其中 @Autowired 只支持设置一个 required 的参数，而 @Resource 支持 7 个参数**，支持的参数如下图所示：
![image.png](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/format%2Cwebp-20250217150542974)

==4）**依赖注入的支持不同**==

@Autowired 和 @Resource 支持依赖注入的用法不同，常见依赖注入有以下 3 种实现：

- ==**属性注入**==
- ==**构造方法注入**==
- ==**Setter 注入**==

其中， ==**@Autowired 支持属性注入、构造方法注入和 Setter 注入，而 @Resource 只支持属性注入和 Setter 注入**==，当使用 @Resource 实现构造方法注入时就会提示以下错误：

![image.png](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/format%2Cwebp-20250217151557333)

**三种注入方法如下:**

**a) 属性注入**

```java
@RestController
public class UserController {
    // 属性注入
    @Autowired
    private UserService userService;

    @RequestMapping("/add")
    public UserInfo add(String username, String password) {
        return userService.add(username, password);
    }
}
```

**b) 构造方法注入**

```java
@RestController
public class UserController {
    // 构造方法注入
    private UserService userService;

    @Autowired
    public UserController(UserService userService) {
        this.userService = userService;
    }

    @RequestMapping("/add")
    public UserInfo add(String username, String password) {
        return userService.add(username, password);
    }
}
```

**c) Setter 注入**

```java
@RestController
public class UserController {
    // Setter 注入
    private UserService userService;

    @Autowired
    public void setUserService(UserService userService) {
        this.userService = userService;
    }

    @RequestMapping("/add")
    public UserInfo add(String username, String password) {
        return userService.add(username, password);
    }
}
```

#### 2.2. @Component,@Repository,@Service, @Controller

我们一般使用 `@Autowired` 注解让 Spring 容器帮我们自动装配 bean。要想把类标识成可用于 `@Autowired` 注解自动装配的 bean 的类,可以采用以下注解实现：

- `@Component`：通用的注解，可标注任意类为 `Spring` 组件。如果一个 Bean 不知道属于哪个层，可以使用`@Component` 注解标注。
- `@Repository` : 对应持久层即 Dao 层，主要用于数据库相关操作。
- `@Service` : 对应服务层，主要涉及一些复杂的逻辑，需要用到 Dao 层。
- `@Controller` : 对应 Spring MVC 控制层，主要用于接受用户请求并调用 Service 层返回数据给前端页面。

#### 2.3 @Configuration和@Bean

**(1) @Configuration的作用**

- **配置类标记**：`@Configuration`用于标记一个类为配置类，其作用相当于XML配置文件。配置类中通过`@Bean`方法定义Bean，表示该类中有一个或多个 `@Bean` 方法,Spring容器会扫描并处理这些方法以构建Bean定义。
  - `@Configuration` 是一个特化版的 `@Component`，它会告诉 Spring 该类是一个配置类，能够包含 Bean 定义方法，并且支持代理机制。
- **代理机制**：配置类会被CGLIB动态代理，确保所有`@Bean`方法返回单例Bean。当配置类中的方法相互调用时，代理会拦截并直接从容器中获取已存在的Bean，避免重复创建。
- **使用场景**：
  - 适合需要复杂初始化逻辑、Bean间依赖管理或外部资源（如数据源）配置的场景
  - **用于配置类的定义，通常与 `@Bean` 一起使用，用来显式声明 Bean**。在 Spring Boot 项目中，它常用于设置应用的配置，或者在传统的 Spring 项目中配置一些自定义 Bean。

**(2) @Bean的作用**

- **定义Bean实例**：`@Bean`用于方法上，表示该方法返回的对象应注册为Spring容器管理的Bean。方法名默认作为Bean的名称，返回值类型作为Bean的类型。
- **灵活配置**：支持指定初始化和销毁方法（`initMethod`、`destroyMethod`），以及作用域（如`@Scope("prototype")`）。例如，可显式配置数据库连接池的生命周期。
- **适用位置**：虽然主要用在`@Configuration`类中，但也可用于`@Component`类（此时无代理机制，可能影响单例行为）。

Eg1:`myService()` 方法返回一个 `MyServiceImpl` 的实例，Spring 会自动将它注册为一个 Bean。

```java
@Configuration
public class AppConfig {

    @Bean
    public MyService myService() {
        return new MyServiceImpl();
    }
}
```

Eg2:在此示例中，`dataSource()`和`service()`均被注册为Bean，且`service()`的依赖通过代理正确注入

```java
@Configuration
public class AppConfig {
    @Bean
    public DataSource dataSource() {
        // 复杂的数据源配置
        return new DruidDataSource();
    }
    @Bean
    public Service service() {
        // 依赖dataSource Bean，代理确保此处调用的是容器中的单例
        return new Service(dataSource());
    }
}

```



#### 2.4. @RestController(老的注解:@Controller)

`@RestController`注解是`@Controller`和`@ResponseBody`的合集,表示这是个控制器 bean,并且是将函数的返回值直接填入 HTTP 响应体中,是 REST 风格的控制器。

*Guide：现在都是前后端分离，说实话我已经很久没有用过`@Controller`。如果你的项目太老了的话，就当我没说。*

单独使用 `@Controller` 不加 `@ResponseBody`的话一般是用在要返回一个视图的情况，这种情况属于比较传统的 Spring MVC 的应用，对应于前后端不分离的情况。`@Controller` +`@ResponseBody` 返回 JSON 或 XML 形式数据

关于`@RestController` 和 `@Controller`的对比，请看这篇文章：[@RestController vs @Controller](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247485544&idx=1&sn=3cc95b88979e28fe3bfe539eb421c6d8&chksm=cea247a3f9d5ceb5e324ff4b8697adc3e828ecf71a3468445e70221cce768d1e722085359907&token=1725092312&lang=zh_CN#rd)。



**(1)`@RestController` 和 `@Controller`的对比**

- **Controller 返回一个页面**

​	单独使用 `@Controller` 不加 `@ResponseBody`的话一般使用在要返回一个视图的情况，这种情况属于比较传统的Spring MVC 的应用，对应于前后端不分离的情况。

![图片](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/640-20250217154248452)

- **@RestController 返回JSON 或 XML 形式数据**

​	但`@RestController`只返回对象，对象数据直接以 JSON 或 XML 形式写入 HTTP 响应(Response)中，这种情况属于 RESTful Web服务，这也是目前日常开发所接触的最常用的情况（前后端分离）。

![图片](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/640-20250217154320668)

- **@Controller +@ResponseBody 返回JSON 或 XML 形式数据**

​	如果你需要在Spring4之前开发 RESTful Web服务的话，你需要使用`@Controller` 并结合`@ResponseBody`注解，也就是说`@Controller` +`@ResponseBody`= `@RestController`（Spring 4 之后新加的注解）。

> `@ResponseBody` 注解的作用是将 `Controller` 的方法返回的对象通过适当的转换器转换为指定的格式之后，写入到HTTP 响应(Response)对象的 body 中，通常用来返回 JSON 或者 XML 数据，返回 JSON 数据的情况比较多。

![图片](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/640-20250217154416109)

- **示例: @Controller+@ResponseBody 返回 JSON 格式数据**

**SpringBoot 默认集成了 jackson ,对于此需求你不需要添加任何相关依赖。**

src/main/java/com/example/demo/controller/Person.java

```java
public class Person {
    private String name;
    private Integer age;
    ......
    省略getter/setter ，有参和无参的construtor方法
}

```

src/main/java/com/example/demo/controller/HelloController.java

```java
@Controller
public class HelloController {
    @PostMapping("/hello")
    @ResponseBody
    public Person greeting(@RequestBody Person person) {
        return person;
    }

}
```

使用 post 请求访问 http://localhost:8080/hello ，body 中附带以下参数,后端会以json 格式将 person 对象返回。

```java
{
    "name": "teamc",
    "age": 1
}
```

- **示例: @RestController 返回 JSON 格式数据**

只需要将`HelloController`改为如下形式：

```java
@RestController
public class HelloController {
    @PostMapping("/hello")
    public Person greeting(@RequestBody Person person) {
        return person;
    }
}
```



#### 2.5. @Scope

声明 Spring Bean 的作用域，使用方法:

```java
@Bean
@Scope("singleton")
public Person personSingleton() {
    return new Person();
}
```

**四种常见的 Spring Bean 的作用域：**

- singleton : 唯一 bean 实例，Spring 中的 bean 默认都是单例的。
- prototype : 每次请求都会创建一个新的 bean 实例。
- request : 每一次 HTTP 请求都会产生一个新的 bean，该 bean 仅在当前 HTTP request 内有效。
- session : 每一个 HTTP Session 会产生一个新的 bean，该 bean 仅在当前 HTTP session 内有效。

#### ==2.6.@PostConstruct==

​	`@PostConstruct` 注解可用于 bean 的任何方法上，但**<font color = '#8D0101'>只会对 bean 的生命周期调用一次</font>**。它通常用于执行一些初始化逻辑，例如建立[数据库](https://cloud.tencent.com/product/tencentdb-catalog?from_column=20065&from=20065)连接、初始化资源等。

==bean初始化顺序：====Constructor(构造方法) -> @Autowired(依赖注入) -> @PostConstruct(注释的方法)==

- **当 Spring 容器完成 Bean 的实例化、完成依赖注入（如 @Resource/@Autowired 注入的属性）之后，@PostConstruct 注解的方法会自动被调用。**

```java
public class MyClass {
    @PostConstruct
    public void init() {
        // 在这里执行初始化操作
    }
}
```

**<u>与构造函数的区别</u>**

- 构造函数：在对象实例化时调用，此时依赖注入尚未完成（如果构造函数中直接使用 jobLogStorage 会得到 null）。
- @PostConstruct：确保依赖注入完成后再执行初始化逻辑，避免 NPE 等错误。

==**eg：**streamis代码==

- init() 方法会在 jobLogStorage 被注入后执行。
- 作用包括：
  - 初始化配置：创建 jobLogBucketConfig 对象（可能用于后续日志存储配置）。
  - 缓存预加载：初始化线程安全的 ConcurrentHashMap 缓存 productNameCache，为后续 getProductName 方法提供缓存容器。
- 使用 @PostConstruct 能保证 jobLogStorage 已注入完毕，后续方法（如 store）调用时不会因未初始化而报错。
- 缓存容器的初始化也需在 Bean 就绪前完成，保证线程安全。

```java
public class DefaultStreamisJobLogService implements StreamisJobLogService {

    @Resource
    private JobLogStorage jobLogStorage;

    private JobLogBucketConfig jobLogBucketConfig;
    private Map<Long,String> productNameCache;

    @PostConstruct
    public void init(){
        jobLogBucketConfig = new JobLogBucketConfig();
        productNameCache = new ConcurrentHashMap<>();
    }
```







### 3. 处理常见的 HTTP 请求类型

**5 种常见的请求类型:**

- **GET**：请求从服务器获取特定资源。举个例子：`GET /users`（获取所有学生）
- **POST**：在服务器上创建一个新的资源。举个例子：`POST /users`（创建学生）
- **PUT**：更新服务器上的资源（客户端提供更新后的整个资源）。举个例子：`PUT /users/12`（更新编号为 12 的学生）
- **DELETE**：从服务器删除特定的资源。举个例子：`DELETE /users/12`（删除编号为 12 的学生）
- **PATCH**：更新服务器上的资源（客户端提供更改的属性，可以看做作是部分更新），使用的比较少，这里就不举例子了。

#### 3.1. GET 请求

**==`@GetMapping("users")` 等价于`@RequestMapping(value="/users",method=RequestMethod.GET)`==**

```java
@GetMapping("/users")
public ResponseEntity<List<User>> getAllUsers() {
 return userRepository.findAll();
}
```

#### 3.2. POST 请求

**==`@PostMapping("users")`等价于`@RequestMapping(value="/users",method=RequestMethod.POST)`==**

关于`@RequestBody`注解的使用，在下面的“前后端传值”这块会讲到。

```java
@PostMapping("/users")
public ResponseEntity<User> createUser(@Valid @RequestBody UserCreateRequest userCreateRequest) {
 return userRepository.save(userCreateRequest);
}
```

**3.3. PUT 请求**

```
@PutMapping("/users/{userId}")` 等价于`@RequestMapping(value="/users/{userId}",method=RequestMethod.PUT)
```

```java
@PutMapping("/users/{userId}")
public ResponseEntity<User> updateUser(@PathVariable(value = "userId") Long userId,
  @Valid @RequestBody UserUpdateRequest userUpdateRequest) {
  ......
}
```

**3.4. DELETE 请求**

```
@DeleteMapping("/users/{userId}")`等价于`@RequestMapping(value="/users/{userId}",method=RequestMethod.DELETE)
```

```java
@DeleteMapping("/users/{userId}")
public ResponseEntity deleteUser(@PathVariable(value = "userId") Long userId){
  ......
}
```

**3.5. PATCH 请求**

一般实际项目中，我们都是 PUT 不够用了之后才用 PATCH 请求去更新数据。

```java
  @PatchMapping("/profile")
  public ResponseEntity updateStudent(@RequestBody StudentUpdateRequest studentUpdateRequest) {
        studentRepository.updateDetail(studentUpdateRequest);
        return ResponseEntity.ok().build();
    }
```



### 4. 前后端传值

**掌握前后端传值的正确姿势，是你开始 CRUD 的第一步！**

#### 4.1. @PathVariable 和 @RequestParam

**==`@PathVariable`用于获取路径参数，`@RequestParam`用于获取查询参数。==**

举个简单的例子：

```java
@GetMapping("/klasses/{klassId}/teachers")
public List<Teacher> getKlassRelatedTeachers(
         @PathVariable("klassId") Long klassId,
         @RequestParam(value = "type", required = false) String type ) {
...
}
```

如果我们请求的 url 是：`/klasses/123456/teachers?type=web`

那么我们服务获取到的数据就是：`klassId=123456,type=web`。

Streams eg:

```java
@RequestMapping(value = "/json/{jobId:\\w+}", method = RequestMethod.GET)
    public Message queryConfig(@PathVariable("jobId") Long jobId, HttpServletRequest request) {
```

#### 4.2. @RequestBody

​	用于读取 Request 请求（可能是 POST,PUT,DELETE,GET 请求）的 body 部分并且==**Content-Type 为 application/json** 格式的数据==，接收到数据之后会自动将数据绑定到 Java 对象上去。**系统会使用`HttpMessageConverter`或者自定义的`HttpMessageConverter`将请求的 body 中的 json 字符串转换为 java 对象**。

我用一个简单的例子来给演示一下基本使用！

我们有一个注册的接口：

```java
@PostMapping("/sign-up")
public ResponseEntity signUp(@RequestBody @Valid UserRegisterRequest userRegisterRequest) {
  userService.save(userRegisterRequest);
  return ResponseEntity.ok().build();
}
```

`UserRegisterRequest`对象：

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class UserRegisterRequest {
    @NotBlank
    private String userName;
    @NotBlank
    private String password;
    @NotBlank
    private String fullName;
}
```

我们发送 post 请求到这个接口，并且 body 携带 JSON 数据：

```json
{ "userName": "coder", "fullName": "shuangkou", "password": "123456" }
```

这样我们的后端就可以直接把 json 格式的数据映射到我们的 `UserRegisterRequest` 类上。

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/%40RequestBody-Ddx5QOeh.png)

👉 **需要注意的是**：**一个请求方法只可以有一个`@RequestBody`，但是可以有多个`@RequestParam`和`@PathVariable`**。 如果你的方法必须要用两个 `@RequestBody`来接受数据的话，大概率是你的数据库设计或者系统设计出问题了！



### 5. 读取配置信息

**很多时候我们需要将一些常用的配置信息比如阿里云 oss、发送短信、微信认证的相关配置信息等等放到配置文件中。**

**下面我们来看一下 Spring 为我们提供了哪些方式帮助我们从配置文件中读取这些配置信息。**

我们的数据源`application.yml`内容如下：

```yaml
wuhan2020: 2020年初武汉爆发了新型冠状病毒，疫情严重，但是，我相信一切都会过去！武汉加油！中国加油！
my-profile:
  name: Guide哥
  email: koushuangbwcx@163.com
library:
  location: 湖北武汉加油中国加油
  books:
    - name: 天才基本法
      description: 二十二岁的林朝夕在父亲确诊阿尔茨海默病这天
    - name: 时间的秩序
      description: 为什么我们记得过去
    - name: 了不起的我
      description: 如何养成一个新习惯？如何让心智变得更成熟？如何拥有高质量的关系？ 如何走出人生的艰难时刻？
```

#### 5.1. @Value(常用)

使用 `@Value("${property}")` 读取比较简单的配置信息：

```
@Value("${wuhan2020}")
String wuhan2020;
```

#### 5.2. @ConfigurationProperties(常用)

通过`@ConfigurationProperties`读取配置信息并与 bean 绑定。

```java
@Component
@ConfigurationProperties(prefix = "library")
class LibraryProperties {
    @NotEmpty
    private String location;
    private List<Book> books;

    @Setter
    @Getter
    @ToString
    static class Book {
        String name;
        String description;
    }
  省略getter/setter
  ......
}
```

你可以像使用普通的 Spring bean 一样，将其注入到类中使用。

### 6. 参数校验

**数据的校验的重要性就不用说了，即使在前端对数据进行校验的情况下，我们还是要对传入后端的数据再进行一遍校验，避免用户绕过浏览器直接通过一些 HTTP 工具直接向后端请求一些违法数据。**

SpringBoot 项目的 spring-boot-starter-web 依赖中已经有 hibernate-validator 包，不需要引用相关依赖。如下图所示（通过 idea 插件—Maven Helper 生成）：

**注**：更新版本的 spring-boot-starter-web 依赖中不再有 hibernate-validator 包（如 2.3.11.RELEASE），需要自己引入 `spring-boot-starter-validation` 依赖

#### 6.1. 验证请求体(RequestBody)

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Person {
  
    @NotNull(message = "classId 不能为空")
    private String classId;
  
    @Size(max = 33)
    @NotNull(message = "name 不能为空")
    private String name;

    @Pattern(regexp = "((^Man$|^Woman$|^UGM$))", message = "sex 值不在可选范围")
    @NotNull(message = "sex 不能为空")
    private String sex;

    @Email(message = "email 格式不正确")
    @NotNull(message = "email 不能为空")
    private String email;

}
```

我们在需要验证的参数上加上了`@Valid`注解，如果验证失败，它将抛出`MethodArgumentNotValidException`。

```java
@RestController
@RequestMapping("/api")
public class PersonController {

    @PostMapping("/person")
    public ResponseEntity<Person> getPerson(@RequestBody @Valid Person person) {
        return ResponseEntity.ok().body(person);
    }
}
```



#### 6.2. 验证请求参数(Path Variables 和 Request Parameters)

**一定一定不要忘记在类上加上 `@Validated` 注解了，这个参数可以告诉 Spring 去校验方法参数。**

```java
@RestController
@RequestMapping("/api")
@Validated
public class PersonController {

    @GetMapping("/person/{id}")
    public ResponseEntity<Integer> getPersonByID(@Valid @PathVariable("id") @Max(value = 5,message = "超过 id 的范围了") Integer id) {
        return ResponseEntity.ok().body(id);
    }
}
```

### 7. JPA 相关

**（1）创建表**

`@Entity`声明一个类对应一个数据库实体。

`@Table` 设置表名

```java
@Entity
@Table(name = "role")
public class Role {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String description;
    省略getter/setter......
}
```

**（2）创建主键**

`@Id`：声明一个字段为主键。

使用`@Id`声明之后，我们还需要定义主键的生成策略。我们可以使用 `@GeneratedValue` 指定主键生成策略。

**1.通过 `@GeneratedValue`直接使用 JPA 内置提供的四种主键生成策略来指定主键生成策略。**

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

**==JPA 使用枚举定义了 4 种常见的主键生成策略，如下：==**

*Guide：枚举替代常量的一种用法*

```java
public enum GenerationType {
    /**
     * 使用一个特定的数据库表格来保存主键
     * 持久化引擎通过关系数据库的一张特定的表格来生成主键,
     */
    TABLE,

    /**
     *在某些数据库中,不支持主键自增长,比如Oracle、PostgreSQL其提供了一种叫做"序列(sequence)"的机制生成主键
     */
    SEQUENCE,

    /**
     * 主键自增长
     */
    IDENTITY,

    /**
     *把主键生成策略交给持久化引擎(persistence engine),
     *持久化引擎会根据数据库在以上三种主键生成 策略中选择其中一种
     */
    AUTO
}
```

一般使用 MySQL 数据库的话，使用`GenerationType.IDENTITY`策略比较普遍一点（分布式系统的话需要另外考虑使用分布式 ID）。

**（3）设置字段类型**

`@Column` 声明字段。

**示例：**

设置属性 userName 对应的数据库字段名为 user_name，长度为 32，非空

```
@Column(name = "user_name", nullable = false, length=32)
private String userName;
```

设置字段类型并且加默认值，这个还是挺常用的。

```
@Column(columnDefinition = "tinyint(1) default 1")
private Boolean enabled;
```

**（4）创建枚举类型的字段**

可以使用枚举类型的字段，不过枚举字段要用`@Enumerated`注解修饰。

```java
public enum Gender {
    MALE("男性"),
    FEMALE("女性");

    private String value;
    Gender(String str){
        value=str;
    }
}
```

```Java
@Entity
@Table(name = "role")
public class Role {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String description;
    @Enumerated(EnumType.STRING)
    private Gender gender;
    省略getter/setter......
}
```

数据库里面对应存储的是 MALE/FEMALE。

**（5）注意**

禁用自动 DDL（如 `spring.jpa.hibernate.ddl-auto=none`），==**必须手动确保 MySQL 字段与 JPA 注解一致**==

#### 关键注解与 MySQL 字段的对应关系

| **JPA 注解**                        | **作用**                                       | **MySQL 字段示例**                         |
| :---------------------------------- | :--------------------------------------------- | :----------------------------------------- |
| `@Id`                               | 标记主键                                       | `id BIGINT PRIMARY KEY`                    |
| `@GeneratedValue`                   | 主键生成策略（如自增）                         | `AUTO_INCREMENT`                           |
| `@Column(name, length, nullable)`   | 定义字段名、长度、是否允许 NULL                | `user_name VARCHAR(50) NOT NULL`           |
| `@Enumerated(EnumType.STRING)`      | 将枚举类型存储为字符串（默认是序号 `ORDINAL`） | ==`user_role VARCHAR(255)`==               |
| `@Temporal(TemporalType.TIMESTAMP)` | 映射日期时间类型                               | `created_at TIMESTAMP`                     |
| `@Lob`                              | 大字段（如文本或二进制数据）                   | `content LONGTEXT` 或 `blob_data LONGBLOB` |

### 8. 事务 @Transactional

在要开启事务的方法上使用`@Transactional`注解即可!

```java
@Transactional(rollbackFor = Exception.class)
public void save() {
  ......
}
```

我们知道 Exception 分为运行时异常 RuntimeException 和非运行时异常。在`@Transactional`注解中如果不配置`rollbackFor`属性,那么事务只会在遇到`RuntimeException`的时候才会回滚,加上`rollbackFor=Exception.class`,可以让事务在遇到非运行时异常时也回滚。

`@Transactional**==` 注解一般可以作用在`类`或者`方法`上。==**

- **作用于类**：当把`@Transactional` 注解放在类上时，表示所有该类的 public 方法都配置相同的事务属性信息。
- **作用于方法**：当类配置了`@Transactional`，方法也配置了`@Transactional`，方法的事务会覆盖类的事务配置信息。

更多关于 Spring 事务的内容请查看我的这篇文章：[可能是最漂亮的 Spring 事务管理详解]() 。



## Spring 事务详解

**什么是事务？**

**事务是逻辑上的一组操作，要么都执行，要么都不执行。**

我们系统的每个业务方法可能包括了多个原子性的数据库操作，比如下面的 `savePerson()` 方法中就有两个原子性的数据库操作。这些原子性的数据库操作是有依赖的，它们要么都执行，要不就都不执行。

```java
  public void savePerson() {
    personDao.save(person);
    personDetailDao.save(personDetail);
  }
```

另外，需要格外注意的是：**事务能否生效数据库引擎是否支持事务是关键**。**比如常用的 MySQL 数据库默认使用支持事务的 `innodb`引擎。但是，如果把数据库引擎变为 `myisam`，那么程序也就不再支持事务了**！

eg：假如小明要给小红转账 1000 元，这个转账会涉及到两个关键操作就是：

> 1. 将小明的余额减少 1000 元。
> 2. 将小红的余额增加 1000 元。

万一在这两个操作之间突然出现错误比如银行系统崩溃或者网络故障，导致小明余额减少而小红的余额没有增加，这样就不对了。事务就是保证这两个关键操作要么都成功，要么都要失败。

```java
public class OrdersService {
  private AccountDao accountDao;
  
  public void setOrdersDao(AccountDao accountDao) {
    this.accountDao = accountDao;
  }

  @Transactional(propagation = Propagation.REQUIRED,
                isolation = Isolation.DEFAULT, readOnly = false, timeout = -1)
  public void accountMoney() {
    //小红账户多1000
    accountDao.addMoney(1000,xiaohong);
    //模拟突然出现的异常，比如银行中可能为突然停电等等
    //如果没有配置事务管理的话会造成，小红账户多了1000而小明账户没有少钱
    int i = 10 / 0;
    //小王账户少1000
    accountDao.reduceMoney(1000,xiaoming);
  }
}
```

另外，数据库事务的 ACID 四大特性是事务的基础，下面简单来了解一下。

### 事务的特性（ACID）

1. **原子性**（`Atomicity`）：事务是最小的执行单位，不允许分割。事务的原子性确保动作要么全部完成，要么完全不起作用；
2. **一致性**（`Consistency`）：执行事务前后，数据保持一致，例如转账业务中，无论事务是否成功，转账者和收款人的总额应该是不变的；
3. **隔离性**（`Isolation`）：并发访问数据库时，一个用户的事务不被其他事务所干扰，各并发事务之间数据库是独立的；
4. **持久性**（`Durability`）：一个事务被提交之后。它对数据库中数据的改变是持久的，即使数据库发生故障也不应该对其有任何影响。

**==这里要额外补充一点==**：**只有保证了事务的持久性、原子性、隔离性之后，一致性才能得到保障。也就是说 A、I、D 是手段，C 是目的！** 

### Spring 事务管理接口介绍

Spring 框架中，事务管理相关最重要的 3 个接口如下：

- ==**`PlatformTransactionManager`**：（平台）事务管理器，Spring 事务策略的核心。==
- ==**`TransactionDefinition`**：事务定义信息(事务隔离级别、传播行为、超时、只读、回滚规则)。==
- ==**`TransactionStatus`**：事务运行状态。==

我们可以把 **`PlatformTransactionManager`** 接口可以被看作是事务上层的管理者，而 **`TransactionDefinition`** 和 **`TransactionStatus`** 这两个接口可以看作是事务的描述。

**`PlatformTransactionManager`** 会根据 **`TransactionDefinition`** 的定义比如事务超时时间、隔离级别、传播行为等来进行事务管理 ，而 **`TransactionStatus`** 接口则提供了一些方法来获取事务相应的状态比如是否新事务、是否可以回滚等等。

**（1）PlatformTransactionManager:事务管理接口**

​	**Spring 并不直接管理事务，而是提供了多种事务管理器** 。Spring 事务管理器的接口是：**`PlatformTransactionManager`** 。主要是因为要将事务管理行为抽象出来，然后不同的平台去实现它，这样我们可以保证提供给外部的行为不变，方便我们扩展。

​	通过这个接口，Spring 为各个平台如：JDBC(`DataSourceTransactionManager`)、Hibernate(`HibernateTransactionManager`)、JPA(`JpaTransactionManager`)等都提供了对应的事务管理器，但是具体的实现就是各个平台自己的事情了。

`PlatformTransactionManager`接口中定义了三个方法：

```java
package org.springframework.transaction;

import org.springframework.lang.Nullable;

public interface PlatformTransactionManager {
    //获得事务
    TransactionStatus getTransaction(@Nullable TransactionDefinition var1) throws TransactionException;
    //提交事务
    void commit(TransactionStatus var1) throws TransactionException;
    //回滚事务
    void rollback(TransactionStatus var1) throws TransactionException;
}
```

**（2）TransactionDefinition:事务属性**

​	事务管理器接口 **`PlatformTransactionManager`** 通过 **`getTransaction(TransactionDefinition definition)`** 方法来得到一个事务，这个方法里面的参数是 **`TransactionDefinition`** 类 ，这个类就定义了一些基本的事务属性。

**什么是事务属性呢？** 事务属性可以理解成事务的一些基本配置，描述了事务策略如何应用到方法上。

事务属性包含了 5 个方面：**隔离级别、传播行为、回滚规则、是否只读、事务超时**

**（3）TransactionStatus:事务状态**

`TransactionStatus`接口用来记录事务的状态 该接口定义了一组方法,用来获取或判断事务的相应状态信息。

`PlatformTransactionManager.getTransaction(…)`方法返回一个 `TransactionStatus` 对象。



#### Spring 支持编程式事务管理和声明式事务管理

**编程式事务管理**

通过 `TransactionTemplate`或者`TransactionManager`手动管理事务，实际应用中很少使用，但是对于你理解 Spring 事务管理原理有帮助。

**声明式事务管理**

推荐使用（代码侵入性最小），实际是通过 AOP 实现（基于`@Transactional` 的全注解方式使用最多）。

使用 **==`@Transactional`注解==**进行事务管理的示例代码如下：

```java
@Transactional(propagation = Propagation.REQUIRED)
public void aMethod {
  //do something
  B b = new B();
  C c = new C();
  b.bMethod();
  c.cMethod();
}
```



### 声明式事务管理`@Transactional`注解的属性详解

#### 事务传播行为

**事务传播行为是为了解决业务层方法之间互相调用的事务问题**。

​	当事务方法被另一个事务方法调用时，必须指定事务应该如何传播。例如：方法可能继续在现有事务中运行，也可能开启一个新事务，并在自己的事务中运行。

- 举个例子：我们在 A 类的`aMethod()`方法中调用了 B 类的 `bMethod()` 方法。这个时候就涉及到业务层方法之间互相调用的事务问题。如果我们的 `bMethod()`如果发生异常需要回滚，如何配置事务传播行为才能让 `aMethod()`也跟着回滚呢？

```java
@Service
Class A {
    @Autowired
    B b;
    @Transactional(propagation = Propagation.xxx)
    public void aMethod {
        //do something
        b.bMethod();
    }
}
@Service
Class B {
    @Transactional(propagation = Propagation.xxx)
    public void bMethod {
       //do something
    }
}
```

在`TransactionDefinition`定义中包括了如下几个表示传播行为的常量：

```java
public interface TransactionDefinition {
    int PROPAGATION_REQUIRED = 0;
    int PROPAGATION_SUPPORTS = 1;
    int PROPAGATION_MANDATORY = 2;
    int PROPAGATION_REQUIRES_NEW = 3;
    int PROPAGATION_NOT_SUPPORTED = 4;
    int PROPAGATION_NEVER = 5;
    int PROPAGATION_NESTED = 6;
    ......
}
```

Spring 相应地定义了一个枚举类：`Propagation`

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250218172012571.png" alt="image-20250218172012571" style="zoom:67%;" />

**正确的事务传播行为可能的值如下**：

**1）`TransactionDefinition.PROPAGATION_REQUIRED`**

使用的最多的一个事务传播行为，我们平时经常使用的`@Transactional`注解默认使用就是这个事务传播行为。**==如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。也就是说==**：

- 如果外部方法没有开启事务的话，`Propagation.REQUIRED`修饰的内部方法会新开启自己的事务，且开启的事务相互独立，互不干扰。
- **如果外部方法开启事务并且被`Propagation.REQUIRED`的话，所有`Propagation.REQUIRED`修饰的内部方法和外部方法均属于同一事务 ，只要一个方法回滚，整个事务均回滚。**

举个例子：如果我们上面的`aMethod()`和`bMethod()`使用的都是`PROPAGATION_REQUIRED`传播行为的话，两者使用的就是同一个事务，只要其中一个方法回滚，整个事务均回滚。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250218172149199.png" alt="image-20250218172149199" style="zoom:80%;" />

**`2）TransactionDefinition.PROPAGATION_REQUIRES_NEW`**

​	创建一个新的事务，如果当前存在事务，则把当前事务挂起。也就是说不管外部方法是否开启事务，`Propagation.REQUIRES_NEW`修饰的内部方法会新开启自己的事务，且开启的事务相互独立，互不干扰。

​	举个例子：如果我们上面的`bMethod()`使用`PROPAGATION_REQUIRES_NEW`事务传播行为修饰，`aMethod`还是用`PROPAGATION_REQUIRED`修饰的话。

- **如果`aMethod()`发生异常回滚，`bMethod()`不会跟着回滚，因为 `bMethod()`开启了独立的事务。**
- **但是，如果 `bMethod()`抛出了未被捕获的异常并且这个异常满足事务回滚规则的话,`aMethod()`同样也会回滚，因为这个异常被 `aMethod()`的事务管理机制检测到了**。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250218172329637.png" alt="image-20250218172329637" style="zoom:75%;" />

**3）`TransactionDefinition.PROPAGATION_NESTED`**:

​	如果当前存在事务，就在嵌套事务内执行；如果当前没有事务，就执行与`TransactionDefinition.PROPAGATION_REQUIRED`类似的操作。也就是说：

- **在外部方法开启事务的情况下，在内部开启一个新的事务，作为嵌套事务存在。**
- **如果外部方法无事务，则单独开启一个事务，与 `PROPAGATION_REQUIRED` 类似。**

这里还是简单举个例子：如果 `bMethod()` 回滚的话，`aMethod()`不会回滚。如果 `aMethod()` 回滚的话，`bMethod()`会回滚。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250218172525156.png" alt="image-20250218172525156" style="zoom:67%;" />

#### 事务隔离级别

`TransactionDefinition` 接口中定义了五个表示隔离级别的常量：

```java
public interface TransactionDefinition {
    ......
    int ISOLATION_DEFAULT = -1;
    int ISOLATION_READ_UNCOMMITTED = 1;
    int ISOLATION_READ_COMMITTED = 2;
    int ISOLATION_REPEATABLE_READ = 4;
    int ISOLATION_SERIALIZABLE = 8;
    ......
}
```

和事务传播行为那块一样，为了方便使用，Spring 也相应地定义了一个枚举类：`Isolation`

```java
public enum Isolation {

  DEFAULT(TransactionDefinition.ISOLATION_DEFAULT),
  READ_UNCOMMITTED(TransactionDefinition.ISOLATION_READ_UNCOMMITTED),
  READ_COMMITTED(TransactionDefinition.ISOLATION_READ_COMMITTED),
  REPEATABLE_READ(TransactionDefinition.ISOLATION_REPEATABLE_READ),
  SERIALIZABLE(TransactionDefinition.ISOLATION_SERIALIZABLE);

  private final int value;
  Isolation(int value) {
    this.value = value;
  }
  public int value() {
    return this.value;
  }
}

```

下面我依次对每一种事务隔离级别进行介绍：

- ==**`TransactionDefinition.ISOLATION_DEFAULT`** :使用后端数据库默认的隔离级别，MySQL 默认采用的 `REPEATABLE_READ` 隔离级别 Oracle 默认采用的 `READ_COMMITTED` 隔离级别.==
- **`TransactionDefinition.ISOLATION_READ_UNCOMMITTED`** :最低的隔离级别，使用这个隔离级别很少，因为它允许读取尚未提交的数据变更，**可能会导致脏读、幻读或不可重复读**
- **`TransactionDefinition.ISOLATION_READ_COMMITTED`** : 允许读取并发事务已经提交的数据，**可以阻止脏读，但是幻读或不可重复读仍有可能发生**
- **`TransactionDefinition.ISOLATION_REPEATABLE_READ`** : 对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，**可以阻止脏读和不可重复读，但幻读仍有可能发生。**
- **`TransactionDefinition.ISOLATION_SERIALIZABLE`** : 最高的隔离级别，完全服从 ACID 的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，**该级别可以防止脏读、不可重复读以及幻读**。但是这将严重影响程序的性能。通常情况下也不会用到该级别。

#### 事务超时属性

**所谓事务超时，就是指一个事务所允许执行的最长时间，如果超过该时间限制但事务还没有完成，则自动回滚事务**。在 `TransactionDefinition` 中以 int 的值来表示超时时间，其单位是秒，默认值为-1，这表示事务的超时时间取决于底层事务系统或者没有超时时间



#### 事务只读属性

```java
package org.springframework.transaction;

import org.springframework.lang.Nullable;

public interface TransactionDefinition {
    ......
    // 返回是否为只读事务，默认值为 false
    boolean isReadOnly();

}
```

​	**对于只有读取数据查询的事务，可以指定事务类型为 readonly，即只读事务**。只读事务不涉及数据的修改，数据库会提供一些优化手段，适合用在有**==多条数据库查询操作==**的方法中。

注意：

- MySQL 默认对每一个新建立的连接都启用了`autocommit`模式。在该模式下，每一个发送到 MySQL 服务器的`sql`语句都会在一个单独的事务中进行处理，执行结束后会自动提交事务，并开启一个新的事务。

- 但是，如果你给方法加上了`Transactional`注解的话，这个方法执行的所有`sql`会被放在一个事务中。如果声明了只读事务的话，数据库就会去优化它的执行，并不会带来其他的什么收益。

- 如果不加`Transactional`，每条`sql`会开启一个单独的事务，中间被其它事务改了数据，都会实时读取到最新值。

分享一下关于事务只读属性，其他人的解答：

- **如果你一次执行单条查询语句，则没有必要启用事务支持，数据库默认支持 SQL 执行期间的读一致性；**
- **如果你一次执行多条查询语句，例如统计查询，报表查询，在这种场景下，多条查询 SQL 必须保证整体的读一致性，==否则，在前条 SQL 查询之后，后条 SQL 查询之前，数据被其他用户改变，则该次整体的统计查询将会出现读数据不一致的状态，此时，应该启用事务支持==**

#### 事务回滚规则

​	这些规则定义了哪些异常会导致事务回滚而哪些不会。默认情况下，事务只有遇到**运行期异常（`RuntimeException` 的子类）时才会回滚，`Error` 也会导致事务回滚，但是，在遇到检查型（Checked）异常时不会回滚**。

如果你想要回滚你定义的特定的异常类型的话，可以这样：

```java
@Transactional(rollbackFor= MyException.class)
```

### @Transactional 注解使用详解

<u>**1）@Transactional 的作用范围**</u>

1. **方法**：推荐将注解使用于方法上，不过需要注意的是：**该注解只能应用到 public 方法上，否则不生效。**
2. **类**：如果这个注解使用在类上的话，表明该注解对该类中所有的 public 方法都生效。
3. **接口**：不推荐在接口上使用。

**`@Transactional` 的常用配置参数总结（只列出了 5 个我平时比较常用的）：**

| 属性名      | 说明                                                         |
| :---------- | :----------------------------------------------------------- |
| propagation | 事务的传播行为，默认值为 REQUIRED，可选的值在上面介绍过      |
| isolation   | 事务的隔离级别，默认值采用 DEFAULT，可选的值在上面介绍过     |
| timeout     | 事务的超时时间，默认值为-1（不会超时）。如果超过该时间限制但事务还没有完成，则自动回滚事务。 |
| readOnly    | 指定事务是否为只读事务，默认值为 false。                     |
| rollbackFor | 用于指定能够触发事务回滚的异常类型，并且可以指定多个异常类型。 |

<u>**2）@Transactional 的使用注意事项总结**</u>

- `@Transactional` 注解只有作用到 **public 方法上事务才生效，不推荐在接口上使用**；
- **避免同一个类中调用 `@Transactional` 注解的方法，这样会导致事务失效**；
- 正确的设置 `@Transactional` 的 `rollbackFor` 和 `propagation` 属性，否则事务可能会回滚失败;
- 被 `@Transactional` 注解的方法所在的类必须被 Spring 管理，否则不生效；
- 底层使用的数据库必须支持事务机制，否则不生效；

<u>==**3）@Transactional 事务注解原理**==</u>

​	**我们知道，`@Transactional` 的工作机制是基于 AOP 实现的，AOP 又是使用动态代理实现的。如果目标对象实现了接口，默认情况下会采用 JDK 的动态代理，如果目标对象没有实现了接口,会使用 CGLIB 动态代理。**

​	如果一个类或者一个类中的 public 方法上被标注`@Transactional` 注解的话，Spring 容器就会在启动的时候为其创建一个代理类，在调用被`@Transactional` 注解的 public 方法的时候，**实际调用的是，`TransactionInterceptor` 类中的 `invoke()`方法**。**这个方法的作用就是在目标方法之前开启事务，方法执行过程中如果遇到异常的时候回滚事务，方法调用完成之后提交事务**。

**<u>4）Spring AOP 自调用问题</u>**

- 当一个方法被标记了`@Transactional` 注解的时候，Spring 事务管理器只会在被其他类方法调用的时候生效，而不会在一个类中方法调用生效。
- 这是因为 Spring AOP 工作原理决定的。因为 Spring AOP 使用**<font color = '#8D0101'>动态代理</font>**来实现事务的管理，它会在运行的时候**<font color = '#8D0101'>为带有 `@Transactional` 注解的方法生成代理对象，并在方法调用的前后应用事物逻辑</font>**。**如果该方法被其他类调用我们的代理对象就会拦截方法调用并处理事务。但是在一个类中的其他方法内部调用的时候，我们代理对象就无法拦截到这个内部调用，因此事务也就失效了**。

eg：

`MyService` 类中的`method1()`调用`method2()`就会导致`method2()`的事务失效。

```java
@Service
public class MyService {

private void method1() {
     method2();
     //......
}
@Transactional
 public void method2() {
     //......
  }
}
```

解决办法就是避免同一类中自调用或者使用 AspectJ 取代 Spring AOP 代理。

## Spring IOC & AOP

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps54.jpg) 

### IOC

​	IOC是Inversion of Control的缩写，多数书籍翻译成“控制反转”。是面向对象编程中的一种设计理念，用来降低程序代码之间的耦合度，它将创建对象以及维护对象之间的关系交给了spring容器进行管理，也就是创建对象的方式反转了。

​	主要分三点：**控制反转、IOC容器和依赖注入**：**<font color = '#8D0101'>在Spring中实现控制反转的是lOC容器，其实现方法是依赖注入(Dependency lnjection,DI)</font>**。

<u>**1）控制反转**</u>

- 没有引入IOC容器之前，对象A依赖于对象B，那么对象A在初始化或者运行到某一点的时候，自己必须主动去创建对象B或者使用已经创建的对象B。无论是创建还是使用对象B，控制权都在自己手上。
- 引入IOC容器之后，对象A与对象B之间失去了直接联系，当对象A运行到需要对象B的时候，IOC容器会主动创建—个对象B注入到对象A需要的地方。

​	通过前后的对比，可以看出来:对象A获得依赖对象B的过程,由主动行为变为了被动行为，控制权颠倒过来了，这就是“控制反转"这个名称的由来。

​	也就是说全部对象的控制权全部给了"第三方"——IOC容器，所以，IOC容器成了整个系统的关键核心，它把系统中的所有对象结合在一起发挥作用，如果没有它，对象与对象之间会彼此失去联系， 

**<u>2）IOC容器</u>**

​	实际上就是个map (key,value)，里面存的是各种对象（在xml里配置的bean节点、@repository,@service、@controller、@component)，在项目启动的时候会读取配置文件里面的bean节点，**根据全限定类名使用反射创建对象放到map里、扫描到有上述注解的类也通过反射创建对象放到map里**。

​	这个时候map里就有各种对象了，接下来我们在代码里需要用到里面的对象时，再通过DI注入(@autowired、@resource等注解，xml里bean节点内的ref属性，项目启动的时候会读取xml节点ref属性根据id注入，也会扫描这些注解，根据类型或id注入;id就是对象名)。

<u>**3）依赖注入**</u>

​	“获得依赖对象的过程被反转了"。控制被反转之后，获得依赖对象的过程由自身管理变为了由IOC容器主动注入。**依赖注入是实现IOC的方法，就是由IOC容器在运行期间，动态地将某种依赖关系注入到对象之中**。

### AOP

#### AOP介绍

​	AOP（Aspect-Oriented Programming，面向切面编程），可以说是OOP（Object-Oriented Programing，面向对象编程）的补充和完善。OOP引入封装、继承和多态性等概念来建立一种对象层次结构，用以模拟公共行为的一个集合。**当我们需要为分散的对象引入公共行为的时候，OOP则显得无能为力。也就是说，OOP允许你定义从上到下的关系，但并不适合定义从左到右的关系**。

​	日志代码往往水平地散布在所有对象层次中，而与它所散布到的对象的核心功能毫无关系。这种散布在各处的无关的代码被称为横切（cross-cutting）代码，在OOP设计中，它导致了大量代码的重复，而不利于各个模块的重用。

​	AOP则利用一种称为“横切”的技术，剖解开封装的对象内部，并**将那些影响了多个类的公共行为封装到一个可重用模块，并将其名为“Aspect”**，即切面。所谓“切面”，简单地说，就是将那些与业务无关却为业务模块所共同调用的功能封装起来，便于减少系统的重复代码，降低模块间的耦合度

​	使用“横切”技术，AOP把软件系统分为两个部分：**核心关注点和横切关注点**。**业务处理的主要流程是核心关注点，与之关系不大的部分是横切关注点**。横切关注点的一个特点是，它们经常发生在核心关注点的多处，而各处都基本相似。比如：权限认证、日志、事务处理。AOP的作用在于分离系统中的各种关注点，将核心关注点和横切关注点分离开来。

#### AOP的基本概念：切面、连接点、切入点等

- 切面（Aspect）：官方的抽象定义为“一个关注点的模块化，这个关注点可能会横切多个对象”。
- 连接点（Joinpoint）：程序执行过程中的某一行为。
- 通知（Advice）：“切面”对于某个“连接点”所产生的动作。
- 切入点（Pointcut）：匹配连接点的断言，在AOP中通知和一个切入点表达式关联。
- 目标对象（Target Object）：被一个或者多个切面所通知的对象。
- AOP代理（AOP Proxy）：在Spring AOP中有两种代理方式，JDK动态代理和CGLIB代理。、

#### AOP 实现打印接口调用时间的原理

**【AOP核心原理】**

​	AOP（Aspect-Oriented Programming）通过 **动态代理** 和 **切面（Aspect）** 实现对目标方法的无侵入式增强。其**核心原理**如下：

1. 动态代理 
   - 如果目标对象实现了接口，Spring 默认使用 **JDK 动态代理** （基于接口生成代理类）。
   - 如果目标对象没有实现接口，Spring 使用 **CGLIB 代理** （通过继承生成子类）。
2. 切面（Aspect） 
   - 封装横切关注点（如日志、性能监控）的模块化组件。
3. 通知（Advice） 
   - 定义切面在何时（如方法执行前、后、环绕）执行。

【**打印接口调用时间的实现原理**】

通过 **环绕通知（@Around）** 在方法执行前后插入时间记录逻辑，计算耗时。

**关键步骤**

1. **定义切面类** ：
   使用 `@Aspect` 注解标记切面类，通过 `@Component` 将其纳入 Spring 容器。
2. **声明切点（Pointcut）** ：
   使用表达式（如 `@annotation` 或 `execution`）指定拦截的目标方法。
3. **环绕通知（@Around）** ：
   在方法执行前后记录时间戳，计算差值得到耗时。

**定义注解（可选）**

- 若需仅对特定接口统计耗时，可自定义注解：

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Timer {
}
```

- 切面类：

```java
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class TimerAspect {

    // 定义切点：拦截所有Controller层的public方法，或带有@Timer注解的方法
    @Pointcut("execution(public * com.example.demo.controller..*.*(..)) || @annotation(Timer)")
    public void pointcut() {}

    @Around("pointcut()")
    public Object measureTime(ProceedingJoinPoint pjp) throws Throwable {
        // 记录开始时间
        long startTime = System.currentTimeMillis();
        
        // 执行目标方法
        Object result = pjp.proceed();
        
        // 计算耗时
        long elapsedTime = System.currentTimeMillis() - startTime;
        
        // 打印日志（可替换为日志框架）
        System.out.println("Method [" + pjp.getSignature().getName() 
            + "] executed in " + elapsedTime + " ms");
        
        return result;
    }
}
```



### Spring AOP动态代理(spring Aop的底层实现原理)

#### 动态代理：

​	静态代理会为每个业务增强都提供了一个代理类，由代理类来创建代理对象。而动态代理并不存在代理类，代理对象由代理代理会生成工具动态生成。

​	AOP思想的实现一般都是基于代理模式，在Java中一般采用JDK动态代理模式，但是我们都知道，JDK动态代理模式只能代理接口而不能代理类。因此，Spring AOP会按照下面两种情况进行切换，因为Spring AOP同时支持CGLIB代理、JDK代理。

#### JDK Proxy

​	如果目标对象的实现类实现了接口，Spring AOP将会采用JDK动态代理来生成AOP代理类；**JDK Proxy使用的是java.lang.reflect包下的代理类来实现，即利用了反射机制来创建代理类对象，<font color = '#8D0101'>JDK动态代理必须要有接口</font>**。

- **Proxy** ：用于生成代理对象。
- **InvocationHandler** ：定义了代理对象调用方法时的行为逻辑。

**实现步骤**

1. **判断目标对象是否实现了接口** ：如果目标对象实现了接口，则使用 JDK 动态代理。
2. **创建代理对象** ：通过 `Proxy.newProxyInstance()` 方法生成代理对象。
3. **增强方法** ：当调用代理对象的方法时，会触发 `InvocationHandler` 的 `invoke()` 方法，从而可以在方法执行前后插入增强逻辑。

#### Cglib Proxy

​	如果**目标对象的实现类没有实现接口**，Spring AOP将会采用CGLIB来生成AOP代理类；Cglib动态代理的原理是**生成目标类的子类**，这个子类对象就是代理对象，代理对象是被增强过的。**注意：不管有没有接口都可以使用CGLIB动态代理，而不是只有在无接口的情况下才使用**。

- **核心类** ：`Enhancer` 是 CGLIB 的核心类，用于生成代理对象

**实现步骤**

1. **判断目标对象是否实现接口** ：如果没有实现接口，则使用 CGLIB 动态代理。
2. **生成代理子类** ：通过 `Enhancer` 创建目标类的子类，并重写其方法。
3. **增强方法** ：在重写的方法中插入增强逻辑。

![image-20250322175420674](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250322175420674.png)

### Spring Bean的生命周期

**生命周期：**

**实例化，属性赋值，初始化，销毁**

![image-20250322175625320](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250322175625320.png)

**作用域@Scope**

声明 Spring Bean 的作用域，使用方法:

```java
@Bean
@Scope("singleton")
public Person personSingleton() {
    return new Person();
}
```

**四种常见的 Spring Bean 的作用域：**

- singleton : 唯一 bean 实例，Spring 中的 bean 默认都是单例的。
- prototype : 每次请求都会创建一个新的 bean 实例。
- request : 每一次 HTTP 请求都会产生一个新的 bean，该 bean 仅在当前 HTTP request 内有效。
- session : 每一个 HTTP Session 会产生一个新的 bean，该 bean 仅在当前 HTTP session 内有效。

**单例bean的线程安全问题**

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps55.jpg)

### Spring如何解决循环依赖

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps56.jpg) 

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps57.jpg) 

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps58.jpg)

### Spring依赖注入的四种方式

在Spring容器中为一个bean配置依赖注入有以下方式：

![image-20250322180533719](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250322180533719.png)

**==看Spring常用注解的@Autowired和@Resource章节==**

### @autowired和@resource的区别

**==看Spring常用注解的@Autowired和@Resource章节==**

## Spring中的设计模式

![image-20250322181107434](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250322181107434.png)

![image-20250322181111154](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250322181111154.png)

## 当userDao存在不止一个bean或没有存在时怎么解决？

![image-20250322181201431](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250322181201431.png)

## BeanFactory和ApplicationContext区别

![image-20250322181443982](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250322181443982.png)

![image-20250322181505578](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250322181505578.png)

## BeanFactory和FactoryBean区别

**BeanFactory是一个接口，public interface BeanFactory**

![image-20250322181539200](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250322181539200.png)

**FactoryBean**

![image-20250322181629122](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250322181629122.png)

![image-20250322181634528](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250322181634528.png)

## SpringMVC

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps59.jpg) 

![image-20250322181927447](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250322181927447.png)

​	**DispatcherServlet，是SpringMVC的核心，实际上是一个HttpServlet，负责将请求纷纷发，所有的请求都经过他来统一分发**。DispatcherSevlet作为前端控制器，整个流程控制的中心，控制其他组件执行，统一调度，降低组件之间的耦合性，提高每个组件的扩展性。