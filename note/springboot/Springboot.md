

# Springboot

1、 简化spring开发，约定大于配置

2、 Starters自动依赖于版本控制

 

Springboot可以添加maven插件后，直接把其打包成jar包，然后java –jar jar包名就可以直接运行

![1566481248972](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\1566481248972.png)

 

## Pom文件探究

他来真正管理springboot版本管理依赖，以后导入的jar包就不用写版本了

 

 ![1566481258741](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\1566481258741.png)

Spring-boot-starter：springboot场景启动器

Springboot将所有功能都抽取出来，做成一个个starter，只需要在相应项目里导入对应的starter就可以了

 

## 原理

### 启动类

 ![1566481268426](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\1566481268426.png)



该注解有@SpringbootConfiguration和@EnableAutoConfiguration

1、@SpringbootConfiguration底层是@Configuration配置类

2、@ EnableAutoConfiguration：开启自动配置，他又包含下面这两个

```java
@AutoConfigurationPackage
@Import({EnableAutoConfigurationImportSelector.class})
```

而其中的@AutoConfiurationPackage有导入了一个Registry，该类用来扫描SpringbootApplication所在包的所有组件，并注册到spring容器

又导入了EnableAutoConfiurationImportSelector类，它将选择要导入的组件（导入很多的自动配置类）注册到容器。

那他是怎么找到这些组件的，启动的时候去类路径下的/META-INF/spring-factories获取自动配置类

 

以前javaee配置文件都有org.springframework.boot.autoconfigure下面的类来代替了

 

 

### 配置文件

推荐使用Application.yml文件

 

要读取配置文件中的内容

1、添加pom

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

2、 添加注解

```
添加@ConfigurationProperties(prefix = "person")
```

3、 把该类注册到容器中

 

 

 

 

### 自动配置

1、 springboot启动时候加载主配置类，开启了自动配置功能

2、 自动配置功能导入了许多的自动配置组件到容器中，用他们来做自动配置

3、以HttpEncodingAutoConfiguration来解释自动原理

```java
@Configuration //一个配置类
@EnableConfigurationProperties({HttpEncodingProperties.class})//启动HttpEncodingProperties类，把这个配置类与该类绑定起来
@ConditionalOnWebApplication //如果当前是web应用，配置类生效
@ConditionalOnClass({CharacterEncodingFilter.class}) //有没有这个类CharacterEncodingFilter（springmvc中乱码解决过滤器），该类从启动HttpEncodingProperties类中获取一些参数
@ConditionalOnProperty(    prefix = "spring.http.encoding",    value = {"enabled"},    matchIfMissing = true) //配置文件是否存在enabled这个属性，及时没有配置，也生效
public class HttpEncodingAutoConfiguration {
```

根据当前不同条件来决定配置类是否生效

4、所有在配置文件中能配置的属性都在xxxxProperties类中封装

```java
@ConfigurationProperties(prefix = "spring.http.encoding")//从配置文件中获取值与属性绑定
public class HttpEncodingProperties {
```



#### 精髓

1. springboot启动会加载大量配置类

2. 我们看我们需要的功能springboot有没有写好的配置类

3. 看这个配置类到底配置了那些组件

4. 给容器中自动配置了添加组件，会从properties文件中获取属性

   XXXAutoConfiguration：自动配置类

   XXXProperties：封装配置文件中相关属性



### Conditional

指定条件成立，才给容器添加组件



### 日志

日志门面：SLF4J

日志实现：log4j JUL和logback



spring默认选择JUL

springboot选用logback



#### SLF4J使用

##### 1.如何在系统中使用

以后开发时候，日志记录方法的调用，不应该直接调用日志实现类，而是调用日志抽象层方法

导入slf4j和log4j logback的jar包

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
public class HelloWorld {  
    public static void main(String[] args) {    
        Logger logger = LoggerFactory.getLogger(HelloWorld.class);    
        logger.info("Hello World");  
    }}
```

##### 2、遗留问题

各个框架底层可能不一样，对应配置也不一样

如何统一？

1. 将系统中其他日志除去
2. 用中间包来代替原有日志
3. 导入slf4j其他的实现



springboot能适配所有日志办法如上面所说

##### 3.使用

springboot默认日志级别info

在配置文件中配置：

logging.level

logging.patten

logging.path





### web

#### 1、springboot对静态资源

```java
@ConfigurationProperties(
    prefix = "spring.resources",
    ignoreUnknownFields = false
)
public class ResourceProperties implements ResourceLoaderAware {
```

可以设置跟资源有关系的配置信息



```java
 public void addResourceHandlers(ResourceHandlerRegistry registry) {
            if (!this.resourceProperties.isAddMappings()) {
                logger.debug("Default resource handling disabled");
            } else {
                Integer cachePeriod = this.resourceProperties.getCachePeriod();
                if (!registry.hasMappingForPattern("/webjars/**")) {
                    this.customizeResourceHandlerRegistration(registry.addResourceHandler(new String[]{"/webjars/**"}).addResourceLocations(new String[]{"classpath:/META-INF/resources/webjars/"}).setCachePeriod(cachePeriod));
                }

                String staticPathPattern = this.mvcProperties.getStaticPathPattern();
                if (!registry.hasMappingForPattern(staticPathPattern)) {
                    this.customizeResourceHandlerRegistration(registry.addResourceHandler(new String[]{staticPathPattern}).addResourceLocations(this.resourceProperties.getStaticLocations()).setCachePeriod(cachePeriod));
                }

            }
        }
```

1. 所有webjars/**，都去classpath:/META-INF/resources/webjars/下找资源，以jar包方式引入

2. 、/**的路径，会从这些文件夹中找资源

   ```
   classpath:/META-INF/resources/"
   "classpath:/resources/"
   "classpath:/static/"
   "classpath:/public/
   / 当前项目根路径
   ```

3. 欢迎页，静态路径中的index.html



#### 2、模板引擎

引入Thymeleaf



语法规则：

1. th:任意html属性，替换原生属性值

2. 表达式 

   ${}获取变量的值

   *{}和${}用法相同，可以配合th:object=${session}    th:text*{user} === th:text=${session.user}

   #{}获取国际化

   @{}获取url

   ~{}片段引用 

#### 3、springmvc自动配置

1. 自动配置了视图解析器，并且可以自定义，只需给容器添加一个实现了ViewResolver的类就可以

2. 自动注册了Converter（转换）、Formatter（格式化）

   Converter：把页面传过来的数据类型转换为pojo对应的数据类型

   ```java
   public void addFormatters(FormatterRegistry registry) {    
       Iterator var2 = this.getBeansOfType(Converter.class).iterator();    	 	     while(var2.hasNext()) {        
           Converter<?, ?> converter = (Converter)var2.next();        registry.addConverter(converter);   
       }   
       var2 = this.getBeansOfType(GenericConverter.class).iterator();    while(var2.hasNext()) {        
           GenericConverter converter = (GenericConverter)var2.next();        registry.addConverter(converter);    
       }    
       var2 = this.getBeansOfType(Formatter.class).iterator();    
       while(var2.hasNext()) {        Formatter<?> formatter = (Formatter)var2.next();        registry.addFormatter(formatter);    }}
   ```

     自己添加格式化器和转换器，只需要添加到容器即可

3. 自动配置ConfigurableWebBindingInlitializer，也可以自定义

   作用：初始化WebDataBinder   请求-===》javaBean 

#### 4、扩展springmvc

编写一个配置类，是WebMvcConfigurerApater类型的，不能标注@EnableWebMvc

```java
@Configuration
public class MyConfig extends WebMvcConfigurerAdapter{
    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        //浏览器发送liquid请求，也1来到succes页面
        registry.addViewController("/liquid").setViewName("success");
    }
}
```

既保留了所有自动配置，也扩展了我们的配置

原理：

1. WebMvcConfigurerApater是springmvc自动配置类

2. 会导入EnableWebMvcConfiguration，他的父类中有如下代码

   ```java
   private final WebMvcConfigurerComposite configurers = new WebMvcConfigurerComposite();
   
       public DelegatingWebMvcConfiguration() {
       }
   	//从容器中获取所有的WebMvcConfigurer
       @Autowired(
           required = false
       )
       public void setConfigurers(List<WebMvcConfigurer> configurers) {
           if (!CollectionUtils.isEmpty(configurers)) {
               this.configurers.addWebMvcConfigurers(configurers);
         }
   
       }
   ```

3. 容器中所有WebMvcConfigurer都会起作用

4. 我们的配置也会被调用

   效果：springmvc自动配置和扩展配置多会起作用



## 开发

### 1、国际化

  步骤：

1. 编写配置文件

2. 配置配置文件地址 

   ```
   spring.messages.basename=i18n.login
   ```

3. 在页面引用配置属性

4. 如果需要在页面点击中文或者英文转换，就需要自己实现LocaleResolver，并添加到容器中

### 2、拦截器

通过拦截器来进行登录检测

1、编写自定义拦截器

```java
public class LoginHandlerInterceptor implements HandlerInterceptor 
```

2、添加拦截器

​	并且定义拦截路径

```java
 //添加拦截器
            @Override
            public void addInterceptors(InterceptorRegistry registry) {
                //springboot已经做好了，不需要静态资源
                registry.addInterceptor(new LoginHandlerInterceptor()).addPathPatterns("/**")
                 .excludePathPatterns("/user/login","/","/index.html");
            }
```

### 3、错误处理

原理：ErrorMvcAutoConfig，错误处理的自动配置

给容器添加这些组件

1、DefaultErrorAttributes

帮我们在页面共享信息

状态码、时间戳、异常信息等

```java
 public Map<String, Object> getErrorAttributes(RequestAttributes requestAttributes, boolean includeStackTrace) {
        Map<String, Object> errorAttributes = new LinkedHashMap();
        errorAttributes.put("timestamp", new Date());
        this.addStatus(errorAttributes, requestAttributes);
        this.addErrorDetails(errorAttributes, requestAttributes, includeStackTrace);
        this.addPath(errorAttributes, requestAttributes);
        return errorAttributes;
    }
```

2、BasicErrorController

```
@Controller
@RequestMapping({"${server.error.path:${error.path:/error}}"})
public class BasicErrorController extends AbstractErrorController {处理默认的/error请求
```

会根据浏览器和客户端返回html和json数据

下面又从ErrorViewResolver得到解析视图

```java
protected ModelAndView resolveErrorView(HttpServletRequest request, HttpServletResponse response, HttpStatus status, Map<String, Object> model) {
        Iterator var5 = this.errorViewResolvers.iterator();

        ModelAndView modelAndView;
        do {
            if (!var5.hasNext()) {
                return null;
            }

            ErrorViewResolver resolver = (ErrorViewResolver)var5.next();
            modelAndView = resolver.resolveErrorView(request, status, model);
        } while(modelAndView == null);

        return modelAndView;
```

3、ErrorPageCustomizer

```java
@Value("${error.path:/error}")
    private String path = "/error";系统出现错误来到error请求
```

4、DefaultErrorViewResolver

去哪个页面是由他来解析得到的，

默认找error/状态码 查找页面

```java
 private ModelAndView resolve(String viewName, Map<String, Object> model) {
        String errorViewName = "error/" + viewName;
     //模板引擎
        TemplateAvailabilityProvider provider = this.templateAvailabilityProviders.getProvider(errorViewName, this.applicationContext);
        return provider != null ? new ModelAndView(errorViewName, model) : 
     this.resolveResource(errorViewName, model);//模板引擎不可用，就去静态资源找
    }
```

步骤：一旦出现4xx等错误，ErrorPageCustomizer就会生效，由BasicErrorController处理异常，具体解析页面由

DefaultErrorViewResolver负责，DefaultErrorAttributes负责提供可以渲染页面的共享对象



自定义错误处理

​      1）、如何定制错误页面

​				1）、you模板引擎情况下，error/状态码，命名错误页面状态码.html，放在error文件夹

​	  2）、如何定制错误json 

​				自适应

```java

@ControllerAdvice
public class MyExceptionHandler {

    @ExceptionHandler(UserNotExistException.class)
    public String handlerException(Exception e){
        Map<String,Object> map = new HashMap<>();
        map.put("code","userNotExits");
        map.put("message",e.getMessage());
        return "forward:/error";
    }

}
```

​      3）、自定义数据携带

​			基础容器的DefaultErrorAttributes对象，重写getErrorAttributes方法



### 4、配置嵌入式servlet

问题？

1）、如何定制和修改servlet相关配置

1、修改和server有关配置，在application.yml文件

2、编写一个EmbeddedServletContainerCustomizer：嵌入式容器定制器

```java
 @Bean
    public EmbeddedServletContainerCustomizer embeddedServletContainerCustomizer(){
        return new EmbeddedServletContainerCustomizer() {
            @Override
            public void customize(ConfigurableEmbeddedServletContainer configurableEmbeddedServletContainer) {
                configurableEmbeddedServletContainer.setPort(8081);
            }
        };
    }
```

注册三大组件

```java
  //注册三大组件
    @Bean
    public ServletRegistrationBean myServlet(){
        ServletRegistrationBean registrationBean  = new ServletRegistrationBean(new MyServlet(),"/myServlet");
        return registrationBean;
    }
```

2）、能不能支持其他servlet容器

排出tomcat的pom，添加其他容器pom

3）、嵌入式容器自动配置原理

```java
@AutoConfigureOrder(-2147483648)
@Configuration
@ConditionalOnWebApplication
@Import({EmbeddedServletContainerAutoConfiguration.BeanPostProcessorsRegistrar.class})
public class EmbeddedServletContainerAutoConfiguration {
    
    //去获取了TomcatEmbeddedServletContainerFactory
    public static class EmbeddedTomcat {
        public EmbeddedTomcat() {
        }

        @Bean
        public TomcatEmbeddedServletContainerFactory tomcatEmbeddedServletContainerFactory() {
            return new TomcatEmbeddedServletContainerFactory();
        }
    }
```

​    获得容器工厂后，去获得容器，并调用容器构造方法，构造方法有调用了initialize()，又调用了tomcat.start()启动tomcat容器

```java
public TomcatEmbeddedServletContainer(Tomcat tomcat, boolean autoStart) {
        this.monitor = new Object();
        this.serviceConnectors = new HashMap();
        Assert.notNull(tomcat, "Tomcat Server must not be null");
        this.tomcat = tomcat;
        this.autoStart = autoStart;
        this.initialize();
    }


 private void initialize() throws EmbeddedServletContainerException {
        logger.info("Tomcat initialized with port(s): " + this.getPortsDescription(false));
        Object var1 = this.monitor;
        synchronized(this.monitor) {
            try {
                this.addInstanceIdToEngineName();

                try {
                    this.removeServiceConnectors();
                    this.tomcat.start();
```

4）、我们如何对配置修改生效

EmbeddedServletContainerAutoConfiguration导入了下面这个内部静态类到容器

该类注册了EmbeddedServletContainerCustomizerBeanPostProcessor到容器中

```java
@Import({EmbeddedServletContainerAutoConfiguration.BeanPostProcessorsRegistrar.class})

  public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
            if (this.beanFactory != null) {
                this.registerSyntheticBeanIfMissing(registry, "embeddedServletContainerCustomizerBeanPostProcessor", EmbeddedServletContainerCustomizerBeanPostProcessor.class);
                this.registerSyntheticBeanIfMissing(registry, "errorPageRegistrarBeanPostProcessor", ErrorPageRegistrarBeanPostProcessor.class);
            }
        }
```

该类是个后置处理器，在bean初始化前后做出改变。会获取所有EmbeddedServletContainerCustomizer，并调用他们的customize，配置生效

```java
 private void postProcessBeforeInitialization(ConfigurableEmbeddedServletContainer bean) {
        Iterator var2 = this.getCustomizers().iterator();

        while(var2.hasNext()) {
            EmbeddedServletContainerCustomizer customizer = (EmbeddedServletContainerCustomizer)var2.next();
            customizer.customize(bean);
        }

    }
```



### 5、容器启动原理

什么时候创建容器工程，什么时候启动容器？

获取嵌入式servlet容器工程：

1、springboot应用启动run方法

2、refreshContext(context) 创建IOC容器

3、refresh(context) spring的ioc容器 重写了onRefresh方法

4、createEmbededServletContainer创建嵌入式servlet容器

5、获取容器工厂，EmbeddedServletContainerCustomizerBeanPostProcessor对容器进行配置

6、使用容器工程创建容器，并启动容器

先启动tomcat容器，再创建ioc容器没有创建出来的bean





### 6、数据访问

如果有数据源springboot自动配置了jdbcTempleate来操作数据库



#### 

#### mybatis注解版

可以使用来批量扫描mapper文件，也可以在mapper文件上加上@Mapper

```java
@MapperScan(value = "com.helloworld.mapper" )
```

并在mapper里面使用@Insert等注解

```java
public interface DepartmentMapper {
    @Select("select * from department where id=#{id}")
    public Department getDepartment(Integer id);

    @Delete("delete from department where id=#{id}")
    public void delete(Integer id);
```



#### mybatis配置文件版

1、编写mybatis-config.xml

2、并编写mapper类和mapper.xml文件

3、在application.yml中配置

```java
mybatis:
  config-location: classpath:mybatis/mybatis-config.xml //指定全局配置文件位置
  mapper-locations: classpath:mybatis/mapper/*.xml
```





### 整合jpa

1、编写一个实体类与数据表进行映射

```java
@Entity //告诉jpa是一个实体类
@Table(name = "tbl_user")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
   private Integer id;

    @Column(name = "last_name",length = 50)
   private String lastName;

    @Column
   private String email;
```

2、编写一个dao接口来操作实体类

```java
//继承JpaRepository
public interface UserRepository extends JpaRepository<User,Integer> {

}
```

3、配置

```
jpa:
    hibernate:
    #更新或者创建表结构
      ddl-auto: update
      #显示sql执行2
    show-sql: true
```

### 7、springboot启动配置原理

重要的几个监听器：

配置在META-INF/spring.factories:

ApplicationContextInitializer //上下文初始化监听器

SpringApplicationListener //spring应用监听器



放在ioc容器中的：

ApplicationRunner

CommandLineRunner



启动流程：

1、创建springbootApplication对象

会调用

```java
 private void initialize(Object[] sources) {
        if (sources != null && sources.length > 0) {
            this.sources.addAll(Arrays.asList(sources));
        }
		//判断是不是web应用
        this.webEnvironment = this.deduceWebEnvironment();
	//从META-INF/spring.factories查找ApplicationContextInitializer并保存下来  this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));
     //查找ApplicationListener并保存下来
        this.setListeners(this.getSpringFactoriesInstances(ApplicationListener.class));
     //查找启动类
        this.mainApplicationClass = this.deduceMainApplicationClass();
    }
```

2、运行run方法

```java
 public ConfigurableApplicationContext run(String... args) {
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        ConfigurableApplicationContext context = null;
        FailureAnalyzers analyzers = null;
        this.configureHeadlessProperty();
     //从META-INF/spring.factories查找SpringApplicationRunListeners
        SpringApplicationRunListeners listeners = this.getRunListeners(args);
     //回调监听器的方法starting()
        listeners.starting();

        try {
            //把控制台输入参数封装成对象
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            //设置环境，并回调SpringApplicationRunListeners environmentPrepared方法
            ConfigurableEnvironment environment = this.prepareEnvironment(listeners, applicationArguments);
            //打印banner图
            Banner printedBanner = this.printBanner(environment);
            //创建ioc容器
            context = this.createApplicationContext();
            new FailureAnalyzers(context);
            //回调之前的ApplicationContextInitializer的inital方法,
            //回调SpringApplicationRunListeners contextLoaded方法
            this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);
            //ioc容器初始化，并且启动tomcat容器
            this.refreshContext(context);
            //对ApplicationRunner和CommandLineRunner进行回调
            this.afterRefresh(context, applicationArguments);
            //回调SpringApplicationRunListeners finished
            listeners.finished(context, (Throwable)null);
            stopWatch.stop();
            if (this.logStartupInfo) {
                (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), stopWatch);
            }
			//启动完毕，返回ioc容器
            return context;
```





### 8、自定义starter

starter：

1、需要使用到的依赖是什么

2、如何编写自动配置

```java
@Configuration //指定配置类
@ConditionalOnMissingBean({WebMvcConfigurationSupport.class}) //指定条件下自动配置生效
@AutoConfigureAfter({DispatcherServletAutoConfiguration.class, ValidationAutoConfiguration.class})  //指定自动配置类顺讯
@Bean //给容器注册组件

@ConfigurationProperties(prefix="") //结合相关xxx.propeties绑定
@EnableConfigurationPropeties //让xxx.propeties自动配置生效,加入到容器

//将需要启动的自动配置类，配置在META-INF/spring.factories中
org.springframework.boot.autoconfigure.EnableAutoConfiguration=

```

3、模式

启动器用来依赖导入

专门来写一个自动配置类库

![1566818646185](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\1566818646185.png)

