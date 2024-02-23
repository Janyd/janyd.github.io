# Dubbo 源码分析（一）配置


> 本文基于 Dubbo 2.7.5 版本

## Dubbo 配置概述
`Dubbo` 提供四种方式配置：
* [`API` 配置](http://dubbo.apache.org/zh-cn/docs/user/configuration/api.html)
* [属性配置](http://dubbo.apache.org/zh-cn/docs/user/configuration/properties.html)
* [`XML` 配置](http://dubbo.apache.org/zh-cn/docs/user/configuration/xml.html)
* [注解配置](http://dubbo.apache.org/zh-cn/docs/user/configuration/annotation.html)

其中包含了许多灵活配置项： 
> FROM  [schema 配置参考手册](http://dubbo.apache.org/zh-cn/docs/user/references/xml/introduction.html)  
> 所有配置项分为三大类，参见下表中的"作用" 一列。  
> * 服务发现：表示该配置项用于服务的注册与发现，目的是让消费方找到提供方。
> * 服务治理：表示该配置项用于治理服务间的关系，或为开发测试提供便利条件。
> * 性能调优：表示该配置项用于调优性能，不同的选项对性能会产生影响。  
> 
> 所有配置最终都将转换为 URL [3] 表示，并由服务提供方生成，经注册中心传递给消费方，各属性对应 URL 的参数，参见配置项一览表中的 "对应URL参数" 列。

## Dubbo 配置一览
![](https://cdn.jsdelivr.net/gh/Janyd/blog-images/imgs/20200911165034.png)  

各个 Config 关系如下  
![](https://cdn.jsdelivr.net/gh/Janyd/blog-images/imgs/20200911165514.png)  
分为4个部分：
* provider-side ：`AbstractServiceConfig`、`ProviderConfig`、`ServiceConfigBase`、`ServiceConfig`、`ServiceBean`、`ProtocolConfig`
* application-shared：`MonitorConfig`、`ModuleConfig`、`ApplicationConfig`、`RegistryConfig`、`MetricsConfig`
* consumer-side：`AbstractReferenceConfig`、`ConsumerConfig`、`ReferenceConfigBase`、`ReferenceConfig`、`ReferenceBean`
* sub-config：`AbstractMethodConfig`、`AbstractInterfaceConfig`、`MethodConfig`、`ArgumentConfig`

## 模型API
`org.apache.dubbo.common.URL`，所有配置类最终会转成`URL`对象，并且由提供者生成经注册中心传递给消费者。
### 属性
```java
public class URL implements Serializable {

    /**
     * 协议
     */
    private final String protocol;

    /**
     * 用户名
     */
    private final String username;

    /**
     * 密码
     */
    private final String password;

    /**
     * 主机地址
     */
    private final String host;

    /**
     * 主机端口
     */
    private final int port;

    /**
     * 路径（服务名）
     */
    private final String path;

    /**
     * 参数集合
     */
    private final Map<String, String> parameters;

//省略...
}
```

## 注解 @Parameter
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
public @interface Parameter {

    /**
     * 配置项名称
     */
    String key() default "";

    /**
     * 是否必须要求
     */
    boolean required() default false;

    /**
     * 是否忽略
     */
    boolean excluded() default false;

    /**
     * 是否需要将配置项进行转义
     */
    boolean escaped() default false;

    /** 
     * 是否为属性
     */
    boolean attribute() default false;

    /**
     * 是否拼接默认属性
     */
    boolean append() default false;

    boolean useKeyAsProperty() default true;
}
```


## 参考链接
[Dubbo API 参考手册](http://dubbo.apache.org/zh-cn/docs/user/references/api.html)
