# Dubbo 源码解析（三）  API配置（服务提供者）


![](https://cdn.jsdelivr.net/gh/Janyd/blog-images/imgs/20200911165514.png) 
## 概述
在这个模块有`AbstractServiceConfig`、`ProviderConfig`、`ServiceConfigBase`、`ServiceConfig`、`ServiceBean`、`ProtocolConfig`


引用官网 [API 配置代码](http://dubbo.apache.org/zh-cn/docs/2.7/user/configuration/api)
```java
import org.apache.dubbo.rpc.config.ApplicationConfig;
import org.apache.dubbo.rpc.config.RegistryConfig;
import org.apache.dubbo.rpc.config.ProviderConfig;
import org.apache.dubbo.rpc.config.ServiceConfig;
import com.xxx.XxxService;
import com.xxx.XxxServiceImpl;
 
// 服务实现
XxxService xxxService = new XxxServiceImpl();
 
// 当前应用配置
ApplicationConfig application = new ApplicationConfig();
application.setName("xxx");
 
// 连接注册中心配置
RegistryConfig registry = new RegistryConfig();
registry.setAddress("10.20.130.230:9090");
registry.setUsername("aaa");
registry.setPassword("bbb");
 
// 服务提供者协议配置
ProtocolConfig protocol = new ProtocolConfig();
protocol.setName("dubbo");
protocol.setPort(12345);
protocol.setThreads(200);
 
// 注意：ServiceConfig为重对象，内部封装了与注册中心的连接，以及开启服务端口
 
// 服务提供者暴露服务配置
ServiceConfig<XxxService> service = new ServiceConfig<XxxService>(); // 此实例很重，封装了与注册中心的连接，请自行缓存，否则可能造成内存和连接泄漏
service.setApplication(application);
service.setRegistry(registry); // 多个注册中心可以用setRegistries()
service.setProtocol(protocol); // 多个协议可以用setProtocols()
service.setInterface(XxxService.class);
service.setRef(xxxService);
service.setVersion("1.0.0");
 
// 暴露及注册服务
service.export();
```
以上是通过 API 来进行将服务者注册到注册中心，并提供服务的。

## ProtocolConfig
`org.apache.dubbo.config.ProtocolConfig`，服务提供者协议配置。
具体属性解释参见[《Dubbo指南 -- dubbo:protocol》](http://dubbo.apache.org/zh-cn/docs/2.7/user/references/xml/dubbo-protocol)

## ProviderConfig
`org.apache.dubbo.config.ProviderConfig`，服务提供配置
具体属性解释参见[《Dubbo指南 -- dubbo:provider》](http://dubbo.apache.org/zh-cn/docs/2.7/user/references/xml/dubbo-provider)

## ServiceConfig
`org.apache.dubbo.config.ServiceConfig`，服务配置
具体属性解释参见[《Dubbo指南 -- dubbo:service》](http://dubbo.apache.org/zh-cn/docs/2.7/user/references/xml/dubbo-service)

## AbstractServiceConfig
`org.apache.dubbo.config.AbstractServiceConfig`，是抽象类，是 `ServiceConfig` 与 `ProviderConfig` 的父类
具体属性结束参见[《Dubbo指南 -- dubbo:service》](http://dubbo.apache.org/zh-cn/docs/2.7/user/references/xml/dubbo-service) 或 [《Dubbo指南 -- dubbo:provider》](http://dubbo.apache.org/zh-cn/docs/2.7/user/references/xml/dubbo-provider)

## AbstractInterfaceConfig
`org.apache.dubbo.config.AbstractInterfaceConfig`，是抽象类，是`AbstractServiceConfig` 与 `AbstractReferenceConfig` 的父类。部分属性看参考上面链接解释。
* `checkRegistry`方法 从`ApplicationConfig`获取注册中心配置，检验并设置到当前配置
* `checkInterfaceAndMethods`方法，检查方法级别配置
* `checkStubAndLocal`方法，检查本地存根与本地实现类型，但local属性已经弃用
属性解释可参考[《Dubbo指南 -- dubbo:service》](http://dubbo.apache.org/zh-cn/docs/2.7/user/references/xml/dubbo-service) 或 [《Dubbo指南 -- dubbo:reference》](http://dubbo.apache.org/zh-cn/docs/2.7/user/references/xml/dubbo-reference)

## AbstractMethodConfig
`org.apache.dubbo.config.AbstractMethodConfig`，抽象方法配置
具体属性解释参见[《Dubbo指南 -- dubbo:method》](http://dubbo.apache.org/zh-cn/docs/2.7/user/references/xml/dubbo-method)

## MethodConfig
`org.apache.dubbo.config.MethodConfig`，方法配置
具体属性解释参见[《Dubbo指南 -- dubbo:method》](http://dubbo.apache.org/zh-cn/docs/2.7/user/references/xml/dubbo-method)

## ServiceBean
`org.apache.dubbo.config.spring.ServiceBean`，服务Bean，继承`ServiceConfig`并实现了 Spring 多个接口，用来将提供者注入到 IOC 容器中
具体属性解释参见[《Dubbo指南 -- dubbo:service》](http://dubbo.apache.org/zh-cn/docs/2.7/user/references/xml/dubbo-service)

## 参考链接
[Dubbo官方指南文档](http://dubbo.apache.org/)
