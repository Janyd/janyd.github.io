# Dubbo 源码分析（四）   配置（服务消费者）


## 概述
在本篇文章介绍 API 配置消费者部分，在此模块中包含有：`AbstractReferenceConfig`、`ConsumerConfig`、`ReferenceConfigBase`、`ReferenceConfig`、`ReferenceBean` 消费者配置类。

## AbstractReferenceConfig
`org.apache.dubbo.config.AbstractReferenceConfig`，抽象引用配置，继承了`AbstractInterfaceConfig`，在上一篇文章也介绍到该抽象类。具体属性可以查看[《Dubbo指南 -- dubbo:reference》](http://dubbo.apache.org/zh-cn/docs/2.7/user/references/xml/dubbo:reference)

## ConsumerConfig
`org.apache.dubbo.config.ConsumerConfig`，消费者配置。继承了 `AbstractReferenceConfig` 。具体属性可以查看[《Dubbo指南 -- dubbo:consumer》](http://dubbo.apache.org/zh-cn/docs/2.7/user/references/xml/dubbo-consumer/)

## ReferenceConfigBase
`org.apache.dubbo.config.ReferenceConfigBase`，引用配置基础。继承了 `AbstractReferenceConfig` 。具体属性可以查看[《Dubbo指南 -- dubbo:reference》](http://dubbo.apache.org/zh-cn/docs/2.7/user/references/xml/dubbo-reference/)

## ReferenceConfigBase
`org.apache.dubbo.config.ReferenceConfigBase`，引用配置基础。继承了 `AbstractReferenceConfig` 。
具体属性可以查看[《Dubbo指南 -- dubbo:reference》](http://dubbo.apache.org/zh-cn/docs/2.7/user/references/xml/dubbo-reference/)

## ReferenceConfig
`org.apache.dubbo.config.ReferenceConfig`，引用配置。继承`ReferenceConfigBase`。
具体属性可以查看[《Dubbo指南 -- dubbo:reference》](http://dubbo.apache.org/zh-cn/docs/2.7/user/references/xml/dubbo-reference/)

## ReferenceBean
`org.apache.dubbo.config.spring.ReferenceBean`，引用Bean，与`ServiceBean`一样为了注入到 IOC 容器中。
* `prepareDubboConfigBeans` 在自动装配 `@Reference` 实例前需要将配置类实例对象初始化完成
具体属性可以查看[《Dubbo指南 -- dubbo:reference》](http://dubbo.apache.org/zh-cn/docs/2.7/user/references/xml/dubbo-reference/)

## 参考链接

