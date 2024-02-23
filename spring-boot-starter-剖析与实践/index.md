# Spring Boot Starter 剖析与实践


## 引言

作为 Java 开发人员来说几乎是离不开 Spring ，Spring 是一个广泛用于开发企业应用程序的开源轻量级框架。但近几年出现了 Spring Boot，构建在传统 Spring 框架之上。因此，提供了完整 Spring 的功能，并且让开发人员更加容易使用。在使用 Spring Boot 缺不了接触到各种 Spring Boot Starter ，比如 `spring-boot-starter-web`，似乎我们只需要将该依赖加入项目中，我们就可以开始开发应用了；还有引入了 `spring-boot-starter-data-jdbc` 后 在配置文件中写上数据库连接信息即可连接数据库，并且可以随意切换数据源组件依赖，Spring Boot Starter 又在其中做了什么适配？我们可否自己实现一个 Spring Boot Starter 呢？本文将会对 Spring Boot Starter 剖析并自行实现一个 Spring Boot Starter 组件。

## 一、Spring Boot Starter 是什么？

Spring Boot Starter 是 Spring Boot 比较重要概念， 是一组方便的依赖描述符，可以将其包含在应用程序中。可以获得所需的所有 Spring 和相关技术的一站式服务，而无需寻找示例代码和复制粘贴依赖描述符负载。在 Spring Boot 中，Starter 是由一组 Maven 依赖项构成的，通常包含一个或者多个自动配置模块 (Auto-Configuration Module) ，这就是 Spring Boot 自动装配。比如我们需要开发 REST 服务，需要依赖 Spring MVC、Tomcat 和 Jackson 等依赖。但是在 Spring Boot 中只需要在项目 Maven `pom.xml`加入以下 Starter 即可：

![](https://storage.360buyimg.com/neos-static-files/fa8e3dca-798b-40b4-bb20-773099def1d8.png)

![](https://storage.360buyimg.com/neos-static-files/f3439b59-8b62-42fb-b362-e8a06ed40858.png)

从上面示例来看，我们使用了相当少的配置创建了一个 REST 应用程序。

## 二、Spring Boot Starter 剖析

我们在前面介绍了 Starter 是什么以及如何快速创建 REST 应用程序，可以看到这只需要简单一个依赖和几行代码即可完成 REST 接口开发，那在没有 Spring Boot 和 Starter 情况下是如何进行开发呢？Spring Boot Starter 工作原理又是如何进行的？接下来让我们分别对纯 Spring 和 Spring Boot Starter 剖析 ，此处我们通过开发 Web 服务与 Dubbo 服务作为例子。

### Spring

#### 环境依赖

* JDK 1.8
* Maven 3
* Tomcat 8（需要依靠 Web 容器服务器才能启动）
* spring-webmvc 4.3.30.RELEASE
* dubbo 2.7.23

#### 开发流程

1. 目录结构与 Maven `demo-service` 依赖内容
   
   <img title="" src="https://storage.360buyimg.com/neos-static-files/48c01353-3a4e-4526-87a2-5aaf1e7312cb.png" alt="" width="604">
   
   ```xml
   <dependencies>
       <!-- SpringMVC -->
       <dependency>
           <groupId>org.springframework</groupId>
           <artifactId>spring-webmvc</artifactId>
           <version>4.3.30.RELEASE</version>
       </dependency>
       <dependency>
           <groupId>javax.servlet</groupId>
           <artifactId>servlet-api</artifactId>
           <version>2.5</version>
       </dependency>
       <!-- 此处需要导入databind包即可， jackson-annotations、jackson-core都不需要显示自己的导入了-->
       <dependency>
           <groupId>com.fasterxml.jackson.core</groupId>
           <artifactId>jackson-databind</artifactId>
           <version>2.9.8</version>
       </dependency>
   
       <!-- Dubbo -->
       <dependency>
           <groupId>org.apache.dubbo</groupId>
           <artifactId>dubbo</artifactId>
           <version>2.7.23</version>
       </dependency>
       <dependency>
           <groupId>org.apache.curator</groupId>
           <artifactId>curator-x-discovery</artifactId>
           <version>5.1.0</version>
       </dependency>
       <dependency>
           <groupId>org.apache.zookeeper</groupId>
           <artifactId>zookeeper</artifactId>
           <version>3.8.0</version>
       </dependency>
   
       <!-- Demo AP -->
       <dependency>
           <groupId>com.demo</groupId>
           <artifactId>demo-api</artifactId>
           <version>1.0-SNAPSHOT</version>
       </dependency>
   </dependencies>
   ```

2. `web/WEB-INF/web.xml` Java Web 配置 SpringMVC 入口
   
   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
            version="4.0">
   
       <!-- Spring监听器 -->
       <listener>
           <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
       </listener>
   
       <context-param>
           <param-name>contextConfigLocation</param-name>
           <param-value>classpath:dubbo.xml</param-value>
       </context-param>
   
       <servlet>
           <servlet-name>springmvc</servlet-name>
           <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
   
           <init-param>
               <param-name>contextConfigLocation</param-name>
               <param-value>classpath:mvc.xml</param-value>
           </init-param>
       </servlet>
   
       <servlet-mapping>
           <servlet-name>springmvc</servlet-name>
           <url-pattern>/</url-pattern>
       </servlet-mapping>
   </web-app>
   ```

3. SpringMVC 配置文件 `mvc.xml` 与 Dubbo 配置文件 `dubbo.xml`
   
   ```xml
   <?xml version="1.0" encoding="utf-8" ?>
   <beans xmlns="http://www.springframework.org/schema/beans"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xmlns:context="http://www.springframework.org/schema/context"
          xmlns:mvc="http://www.springframework.org/schema/mvc"
          xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc.xsd">
   
       <context:component-scan base-package="com.demo.controller"/>
   
       <!-- 开启 MVC 注解驱动 -->
       <mvc:annotation-driven/>
   
       <!-- 访问静态资源 -->
       <mvc:default-servlet-handler/>
   </beans>
   ```
   
   ```xml
   <?xml version="1.0" encoding="utf-8" ?>
   <beans xmlns="http://www.springframework.org/schema/beans" xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
   
       <!-- Dubbo -->
       <dubbo:application name="demo-service"/>
       <dubbo:registry address="zookeeper://127.0.0.1:2181"/>
       <dubbo:protocol name="dubbo" port="20880"/>
       <bean id="demoServiceImpl" class="com.demo.provider.DemoServiceImpl"/>
       <dubbo:service interface="com.demo.api.DemoService" ref="demoServiceImpl"/>
   </beans>
   ```

4. 编写 Controller 接口与 Dubbo RPC 接口
   
   ```java
   package com.demo.controller;
   
   import org.springframework.web.bind.annotation.GetMapping;
   import org.springframework.web.bind.annotation.RestController;
   
   @RestController
   public class HelloController {
   
       @GetMapping(value = "/say/hello")
       public HelloEntity sayHello() {
           return new HelloEntity("Hello World");
       }
   
   }
   ```
   
   ```java
   package com.demo.provider;
   
   import com.demo.api.DemoService;
   import com.demo.dto.HelloEntity;
   
   public class DemoServiceImpl implements DemoService {
   
       @Override
       public HelloEntity sayHello() {
           return new HelloEntity("Hello World");
       }
   }
   ```

5. 以上还无法单独运行，需要将以上打包成 `war` 包放入到 Tomcat 才可运行。

#### 剖析

从上面开发流程可以看到入口都在 `web.xml` 中，可以看到有一个监听器与一个 Servlet ，并且有初始化参数有 `dubbo.xml` 与 `mvc.xml`
在 Spring Boot 出现之前 Spring 都是以 XML 为配置方式描述 Bean 或者在 XML 配置注解驱动与上下文扫描方式解析 Bean的，所以我们从这里可以看出这里有两个 XML 文件。经过分析源代码整理出以下 XML 标签解析到 Bean 解析流程，如下：
![](https://storage.360buyimg.com/neos-static-files/5e3afccf-abd7-4a21-bd0b-09e66fd8dc9b.png)

1. 由 Tomcat 启动加载 `web.xml` 并通过监听器和 Servlet 让 Spring 加载 XML 并解析。
2. 直到 `BeanDefinitionParserDelegate#parseCustomElement` 开始解析自、定义标签，找到 `mvc:xxx` 或 `dubbo:xxx` 标签找到了 XML 命名空间。
3. `DefaultNamespaceHandlerResolver` 处理逻辑：以懒加载方式加载所有 jar 中 `META-INF/spring.handlers` （路径必须得是这个）并缓存到 `handlerMappings` ，通过命名空间 URI 找到与之对应的处理类，SpringMVC 与 Dubbo 命名空间处理类分别为 `MvcNamespaceHandler` 和 `DubboNamespaceHandler`
4. `MvcNamespaceHandler` 和 `DubboNamespaceHandler` 都分别实现了 `NamespaceHandler#init` 方法，内容如下：
   ![](https://storage.360buyimg.com/neos-static-files/6f02226a-7c12-4f8a-aa62-29533c3669fc.png)
   ![](https://storage.360buyimg.com/neos-static-files/55691ed9-a48f-4fd5-8bd1-63791b1a3a2c.png)
    init 方法都注册了 SpringMVC 与 Dubbo 标签对应的 `BeanDefinitionParser` 到 `NamespaceHandlerSupport#parsers`，也是在上一步 `DefaultNamespaceHandlerResolver` 根据标签获取到该标签的 `BeanDefinitionParser` 从而将对应的 Bean 注册到 Spring IOC 容器里，注册逻辑不是本文内容这里就不多讲，至此 SpringMVC 与 Dubbo 加载流程已完成。

从以上加载流程可以看出，在没有 Spring Boot 之前，Spring 主要都是围绕着 XML 配置来启动，同时加载 XML 自定义标签找到对应命名空间，并且扫描找到 classpath 下 `META-INF/spring.handlers` 找到命名空间处理类来去解析当前标签。

### Spring Boot

#### 环境依赖

* JDK 1.8
* Maven 3
* spring-boot 2.6.9
* dubbo 2.7.23

#### 开发流程

1. 目录结构与 Maven `demo-spring-boot` 依赖内容
   ![](https://storage.360buyimg.com/neos-static-files/ed059881-6e1f-4e3d-9b90-1ed4eaeaa9c8.png)
   
   ```xml
   <dependencies>
       <dependency>
           <groupId>org.springframework.boot</groupId>
           <artifactId>spring-boot-starter-web</artifactId>
       </dependency>
   
       <!-- dubbo -->
       <dependency>
           <groupId>org.apache.dubbo</groupId>
           <artifactId>dubbo-spring-boot-starter</artifactId>
           <version>2.7.23</version>
       </dependency>
       <dependency>
           <groupId>org.apache.curator</groupId>
           <artifactId>curator-x-discovery</artifactId>
           <version>5.1.0</version>
       </dependency>
       <dependency>
           <groupId>org.apache.zookeeper</groupId>
           <artifactId>zookeeper</artifactId>
           <version>3.8.0</version>
       </dependency>
   
       <dependency>
           <groupId>com.demo</groupId>
           <artifactId>demo-api</artifactId>
           <version>1.0-SNAPSHOT</version>
       </dependency>
   </dependencies>
   ```

2. 应用程序入口 `Application`，`@EnableDubbo` 开启 `Dubbo` 组件
   
   ```java
   @SpringBootApplication
   public class DemoSpringBootApplication {
   
       public static void main(String[] args) {
           SpringApplication.run(DemoSpringBootApplication.class, args);
       }
   
   }
   ```

3. `application.yml` 文件内容
   
   ```yml
   dubbo:
     application:
       name: demo-provider
     protocol:
       port: 20880
       name: dubbo
     registry:
       address: zookeeper://127.0.0.1:2181
   ```

4. 编写 Controller 接口与 Dubbo RPC 接口
   
   ```java
   package com.demo.controller;
   
   import com.demo.dto.HelloEntity;
   import org.springframework.web.bind.annotation.GetMapping;
   import org.springframework.web.bind.annotation.RestController;
   
   @RestController
   public class HelloController {
   
       @GetMapping(value = "/say/hello")
       public HelloEntity sayHello() {
           return new HelloEntity("Hello World");
       }
   
   }
   ```
   
   ```java
   package com.demo.provider;
   
   import com.demo.api.DemoService;
   import com.demo.dto.HelloEntity;
   
   @DubboService 
   public class DemoServiceImpl implements DemoService {
   
       @Override
       public HelloEntity sayHello() {
           return new HelloEntity("Hello World");
       }
   }
   ```

5. 由于 `spring-boot-starter-web` 已经内嵌 tomcat ，只需要直接运行 `DemoSpringBootApplication#main`  方法即可运行应用

#### 剖析

从开发流程上没办法第一时间找到解析入口，唯一入口就是在 `DemoSpringBootApplication` ，经过源代码分析得出以下流程：

![](https://storage.360buyimg.com/neos-static-files/52e7f4af-a3a9-4474-8651-6bc5a12d0604.png)

1. 应用 `DemoSpringBootApplication` 类上有 `@SpringBootApplication` 注解，而该注解由以下三个注解组成：
   
   * `@SpringBootConfiguration`，标注当前类为一个配置类，与 `@Configuration` 注解功能一致 ，被 `@Configuration` 注解的类对应 Spring 的 XML 版的容器。
   
   * `@EnableAutoConfiguration`，开启启动自动装配的关键，由 `@AutoConfigurationPackage` 与 `@Import(AutoConfigurationImportSelector.class)`组成
   
   * `@ComponentScan`，按照当前类路径扫描含有 `@Service`、`@Controller`等等注解的类，等同于 Spring XML 中的 `context:component-scan`。

2. Spring Boot 自动装配由 `@EnableAutoConfiguration` 导入的 `AutoConfigurationImportSelector` 类，会调用 `SpringFactoriesLoader#loadFactoryNames` 从 ClassPath 下扫描所有 jar 包的 `META-INF/spring.factories` 内容，由于传入的 `EnableAutoConfiguration.class`，只会返回 `org.springframework.boot.autoconfigure.EnableAutoConfiguration` key 的值，得到一个全限定类名字符串数组 `configurations`。
   
   ![](https://storage.360buyimg.com/neos-static-files/607a86f4-c1b1-4025-88f9-bdcaf4080447.png)

3. `configurations` 经过去重与声明式排除后，会进行以下进行过滤自动装配：
   
   ```java
   configurations = getConfigurationClassFilter().filter(configurations)
   ```
   
   分成两部分：获取过滤器和执行过滤。
   
   * `getConfigurationClassFilter()`，也是通过`SpringFactoriesLoader#loadFactoryNames` 在`META-INF/spring.factories` 找到 Key 为 `org.springframework.boot.autoconfigure.AutoConfigurationImportFilter`的值，目前只有：`OnBeanCondition`、`OnClassCondition`、`OnClassCondition` 三个过滤器。
   
   * 执行过滤，会根据配置类上含有 `@ConditionOnBean`、`@ConditionalOnClass`、`@ConditionalOnWebApplication` 等等条件注解来过滤掉部分配置类。比如 `WebMvcAutoConfiguration` 指定需要在 `@ConditionOnWebApplication` 下才生效。
     
     ![](https://storage.360buyimg.com/neos-static-files/b5664b75-46d9-40f2-a896-b93b1c08e614.png)

4. 在引入各类 Configuration 的配置类后，配置类结合 `@Bean` 来完成 Spring Bean 解析和注入，同时 Spring Boot 还提供了许多 `@ConditionalXXX` 给开发者完成灵活注入。

以上就是 Spring Boot 自动装配过程，Spring Boot 利用被 `@Configuration` 注解的配置类来代替 Spring XML 完成 Bean 的注入，然后由 `SpringFactoriesLoader` 最终加载 `META-INF/spring.factories` 中的自动配置类实现自动装配过程，依靠约定大于配置的思想，如果开发的 Starter 想要被生效就需要按照 Spring Boot 的约定。

### 小结

通过 Spring 与 Spring Boot 开发流程对比，可以看到 Spring Boot 使用相当少的代码和配置完成了 Web 与 Dubbo 独立应用开发，这里也是得益于 Spring Boot Starter 自动装配的能力，自动配置是 Spring Boot 的主要功能。通过消除定义一些属于自动配置类一部分的 Bean 的需要自动配置可以帮忙加速和简单的开发

#### SPI

我们从上面剖析发现，两者都使用了一项机制去加载引入的 jar 包中的配置文件加载对应类，那就是 **SPI** （Service Provider Interface）

SPI （Service Provider Interface）， 是 Java 内置的一种服务提供发现机制，提高框架的扩展性。

![](https://storage.360buyimg.com/neos-static-files/d3a500ff-c73d-4bed-aa37-8e9947a0009b.png)

##### Java SPI

Java 内置的 SPI 通过 `java.util.ServiceLoader` 类解析 Classpath 和 jar 包的 `META-INF/services` 目录下的以接口全限定名命名的文件，并加载该文件中指定的接口实现类，以此完成调用。

但是 Java SPI 会有一定不足：

* 不能做到按需加载，需要遍历所有的实现并实例化，然后在循环中找到所需要的实现。

* 多个并发多线程使用 `ServiceLoader` 类的实例不安全

* 加载不到实现类时抛出并不是真正原因的异常，错误难定位。

##### Spring SPI

Spring SPI 沿用了 Java SPI ，但是在实现上和 Java SPI 存在差异，但是核心机制相同，在不修改 Spring 源码前提下，可以做到对 Spring 框架的扩展开发。

* 在 Spring XML 中，由 `DefaultNamespaceHandlerResolver` 负责解析 `spring.handlers`生成 namespaceUri 和 NamespaceHandler 名称的映射，等有需要时在进行实例化。

* 在 Spring Boot 中，由 `SpringFactoriesLoader` 负责解析 `spring.factories` 文件，并将指定接口的所有实现类/全限定类名返回。

#### Spring Boot 2.7.0

在本文中 Spring Boot 自动装配使用了 SPI 来加载到 `EnableAutoConfiguration` 所指定的自动装配的类名，但在 Spring Boot **2.7.0** 之后自动装配 SPI 机制有所改动，`META-INF/spring.factories` 将废弃，同时在 Spring Boot 3 以上会将相关代码移除，；改动如下：

* 新的注解：`@AutoConfiguration` 代替 `@Configuration`

* 读取自动装配的类文件位置改为：`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`，并且实现类全限定类名按照一行一个

* 由 `org.springframework.boot.context.annotation.ImportCandidates#load` 负责解析 `META-INF/spring/%s.imports` ，其中 `%s` 是接口名的占位符

## 三、Spring Boot Stater 实践

在使用 `spring-boot-starter-jdbc` 或者 `spring-boot-starter-jpa` 等数据库操作时，通常会引入一个数据库数据源连接池，比如：`HikariCP`、`DBCP`等，同时可随意切换依赖而不需要去更改任何业务代码，开发人员也无需关注底层实现，在此我们自定义一个 Starter 同时也实现这种功能。因为我们以开发一个分布式锁的 Starter 并拥有多个实现：Zookeeper、Redis。 在此使用 Spring Boot 2.6.9 版本。

### 开发

#### 项目结构与 Maven 依赖

```
└── src
    ├── main
    │   ├── java
    │   │   └── com.demo.distributed.lock
    │   │      ├── api
    │   │      │   ├── DistributedLock.java
    │   │      │   └── LockInfo.java
    │   │      ├── autoconfigure
    │   │      │   ├── DistributedLockAutoConfiguration.java
    │   │      │   └── DistributedLockProperties.java
    │   │      ├── redis
    │   │      │   └── RedisDistributedLockImpl.java
    │   │      └── zookeeper
    │   │          └── ZookeeperDistributedLockImpl.java
    │   └── resources
    │       └── META-INF
    │           └── spring.factories
```

```xml
<dependencies>
    <!-- Spring Boot 自动装配注解 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-autoconfigure</artifactId>
    </dependency>

    <!-- 生成 META-INF/spring-configuration-metadata.json -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-configuration-processor</artifactId>
        <optional>true</optional>
    </dependency>

    
    <!-- Zookeeper -->
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-framework</artifactId>
        <version>5.1.0</version>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-recipes</artifactId>
        <version>5.1.0</version>
        <scope>provided</scope>
    </dependency>

    <!-- Redis -->
    <dependency>
        <groupId>org.redisson</groupId>
        <artifactId>redisson</artifactId>
        <version>3.23.1</version>
        <scope>provided</scope>
    </dependency>
</dependencies>
```

在依赖里可以看到 Zookeeper 和 Redis 依赖关系被设置为 `providerd` ，作用为编译与测试阶段使用，不会随着项目一起发布。即打包时不会带上该依赖。该设置在 Spring Boot Starter 作用较大，后面会提到。

#### 分布式锁接口与实现

##### 接口

```java
public interface DistributedLock {

    /**
     * 加锁
     */
    LockInfo tryLock(String key, long expire, long waitTime);

    /**
     * 释放锁
     */
    boolean release(LockInfo lock);

}
```

##### Redis 实现

```java
public class RedisDistributedLockImpl implements DistributedLock {

    private final RedissonClient client;

    public RedisDistributedLockImpl(RedissonClient client) {
        this.client = client;
    }

    @Override
    public LockInfo tryLock(String key, long expire, long waitTime) {
        //do something
        return null;
    }

    @Override
    public boolean release(LockInfo lock) {
        //do something
        return true;
    }
}
```

##### Zookeeper 实现

```java
public class ZookeeperDistributedLockImpl implements DistributedLock {

    private final CuratorFramework client;

    public ZookeeperDistributedLockImpl(CuratorFramework client) {
        this.client = client;
    }

    @Override
    public LockInfo tryLock(String key, long expire, long waitTime) {
        return null;
    }

    @Override
    public boolean release(LockInfo lock) {
        return false;
    }
} 
```

#### DistributedLockAutoConfiguration 配置类

```java
@EnableConfigurationProperties(DistributedLockProperties.class)
@Import({DistributedLockAutoConfiguration.Zookeeper.class, DistributedLockAutoConfiguration.Redis.class})
public class DistributedLockAutoConfiguration {

    @Configuration
    @ConditionalOnClass(CuratorFramework.class)
    @ConditionalOnMissingBean(DistributedLock.class)
    @ConditionalOnProperty(name = "distributed.lock.type", havingValue = "zookeeper",
            matchIfMissing = true)
    static class Zookeeper {

        @Bean
        CuratorFramework curatorFramework(DistributedLockProperties properties) {
            //build CuratorFramework client
            return null;
        }


        @Bean
        ZookeeperDistributedLockImpl zookeeperDistributedLock(CuratorFramework client) {
            return new ZookeeperDistributedLockImpl(client);
        }
    }


    @Configuration
    @ConditionalOnClass(RedissonClient.class)
    @ConditionalOnMissingBean(DistributedLock.class)
    @ConditionalOnProperty(name = "distributed.lock.type", havingValue = "redis",
            matchIfMissing = true)
    static class Redis {

        @Bean
        RedissonClient redissonClient(DistributedLockProperties properties) {
            //build RedissonClient client
            return null;
        }

        @Bean
        RedisDistributedLockImpl redisDistributedLock(RedissonClient client) {
            return new RedisDistributedLockImpl(client);
        }
    }
}

```

* `@EnableConfigurationProperties(DistributedLockProperties.class)` 开启配置类 Properties 信息，会将配置文件里的信息注入 Properties 类里。

* `@Configuration` 配置注解

* `@ConditionalOnClass(CuratorFramework.class)` 条件注解，要求存在 `CuratorFramework` 类当前配置类才生效，Redis 的子配置类同理。

* `@ConditionalOnMissingBean(DistributedLock.class)` 条件注解，Spring 不存在 `DistributedLock` Bean 当前配置类才生效。

* `@ConditionalOnProperty(name = "distributed.lock.type", havingValue = "zookeeper",
   matchIfMissing = true)` 条件注解，这里判断配置文件 `distributed.lock.type` 等于 `zookeeper` 才生效，当如果没配置则默认当做 `zookeeper`

* `@Bean` 将方法返回的 Bean 注入到 Spring IOC 容器里，方法入参中含依赖的 Bean 

#### spring.factories

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.demo.distributed.lock.autoconfigure.DistributedLockAutoConfiguration
```

我们只需要将该文件放到 `resource/META-INF/spring.factories` 下，就会被 Spring Boot 加载，这也是 Spring Boot 的约定大于配置的思想。

### 使用

#### Maven 依赖关系

```xml

<dependencies>
    <dependency>
        <groupId>com.demo</groupId>
        <artifactId>distributed-lock-spring-boot-starter</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </dependency>
</dependencies>

<profiles>
    <profile>
        <id>dev</id>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
        <dependencies>
            <!-- Redis -->
            <dependency>
                <groupId>org.redisson</groupId>
                <artifactId>redisson</artifactId>
                <version>3.23.1</version>
            </dependency>
        </dependencies>
    </profile>
    <profile>
        <id>test</id>
        <dependencies>
            <dependency>
                <groupId>org.apache.curator</groupId>
                <artifactId>curator-framework</artifactId>
                <version>5.1.0</version>
            </dependency>
            <dependency>
                <groupId>org.apache.curator</groupId>
                <artifactId>curator-recipes</artifactId>
                <version>5.1.0</version>
            </dependency>
        </dependencies>
    </profile>
</profiles>
```

此处结合 Maven profile 功能按照不同环境依赖不同分布式锁底层实现，同时 Spring Boot 也提供了 Spring Boot Profile 加载不同配置，可以从开发、测试、生产环境使用不同底层了，同时 Maven profile 可以根据 `-P` 指定加载不同的依赖进行打包，解决了不同环境使用不同分布式锁实现。

#### 代码使用

```java
private final DistributedLock distributedLock;

public DemoServiceImpl(DistributedLock distributedLock) {
    this.distributedLock = distributedLock;
}

public void test() {
    LockInfo lock = null;
    try {
        lock = distributedLock.tryLock("demo", 1000, 1000);
        //do something
    } finally {
        if (lock != null) {
            distributedLock.release(lock);
        }
    }
}
```

业务代码中由于依赖的是接口，结合 Spring Boot Starter + Maven Profile 不管依赖哪个分布式锁实现，都无需去修改代码。

## 四、总结

通过本文 Spring 与 Spring Boot 对比，在开发流程上，Spring Boot 相对 Spring XML 减少了很多代码，同时依赖关系上我们只需要引入一个即可；在工作原理上，虽然都使用了 SPI 机制，实现上略有不同，但目标都是保留了扩展性。 通过自定义一个 Spring Boot Starter 了解到 Spring Boot 的约定大于配置思想，同时可以结合 Maven Profile 机制我们可以在不同环境使用不同的底层实现。

