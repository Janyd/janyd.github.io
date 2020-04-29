# Mybatis加载流程分析（一）- 开篇


## Mybatis是什么？
官方解释：
> [MyBatis](https://mybatis.org/mybatis-3/zh/index.html) 是一款优秀的持久层框架，它支持自定义 SQL、存储过程以及高级映射。MyBatis 免除了几乎所有的 JDBC 代码以及设置参数和获取结果集的工作。MyBatis 可以通过简单的 XML 或注解来配置和映射原始类型、接口和 Java POJO（Plain Old Java Objects，普通老式 Java 对象）为数据库中的记录。 

其实就是一款与数据库打交道的框架，与单纯使用JDBC去连接数据库相比，框架能够给予的东西还是蛮多的，例如：自动打开连接、事务管理、动态SQL等等。  

既然有这么优秀的框架，那么我们来分析一下Mybatis的源码与原理，在接下来的文章会从按流程来分析，遇到Mybatis的基础组件会展开分析，也就是以流程为主基础组件为辅的结构去解析源码，在文章的后面也会附上一些参考文章供大家深入的了解

## Mybatis加载流程
{{< mermaid >}}
sequenceDiagram
    participant U as User
    participant S as SqlSessionFactoryBuilder
    participant X as XMLConfigBuilder
    U->>+S: 指定配置文件
    S->>+X: 传入配置文件位置
    X-->>-S: 加载解析并构建Configuration
    S-->>-U: 构建SqlSessionFactory
{{< /mermaid >}}

从简易的时序图看出整个加载流程主要的为了能够构建出`Configuration`，从而构建`SqlSessionFactory`，`Configuration`是个重量级配置类，也是Mybatis框架核心配置，几乎贯穿了整个框架，而构建`Configuration`是在`XMLConfigBuilder`完成的，所以本篇文章主要是针对该类如何加载与解析配置文件，大概流程如下：
1. 加载配置文件`mybatis-config.xml`(也可以不是这个文件名，在下文称配置文件)，以下都是解析配置文件中的节点
2. 解析`<properties />`，解析出动态配置
3. 解析`<settings />`节点，
    - 加载用户自定义VFS实现类
    - 加载用户自定义日志实现类
4. 解析`<typeAliases />`，该配置是指定别名对应的类型
5. 解析`<plugins />`，加载用户自定义的插件
6. 加载`<objectFactory />`，用户自定义的对象工厂类
7. 加载`<objectWrapperFactory />`，用户自定义的对象包装工厂
8. 加载`<reflectorFactory />`，用户自定义的反射器工厂
9. 将settings得到的配置全部塞进`Configuration`对象
10. 加载`<environments />`环境配置信息
11. 加载`<databaseIdProvider />` 数据库厂商标识
12. 加载`<typeHandlers />`用户自定义的typeHandler
13. 加载`<mappers />`加载每个Mapper.xml文件  

以上流程可从`XMLConfigBuilder#parseConfiguration`获得，代码如下：
```Java
private void parseConfiguration(XNode root) {
        try {
            // issue #117 read properties first
            //解析properties节点
            propertiesElement(root.evalNode("properties"));
            Properties settings = settingsAsProperties(root.evalNode("settings"));
            //加载VFS实现类
            loadCustomVfs(settings);
            //加载日志实现类
            loadCustomLogImpl(settings);
            //加载别名
            typeAliasesElement(root.evalNode("typeAliases"));
            //加载插件
            pluginElement(root.evalNode("plugins"));
            //对象工厂实现类
            objectFactoryElement(root.evalNode("objectFactory"));
            //对象包装工厂类
            objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
            //Reflector工厂实现类
            reflectorFactoryElement(root.evalNode("reflectorFactory"));
            //settings配置设置进configuration对象里
            settingsElement(settings);
            // read it after objectFactory and objectWrapperFactory issue #631
            //加载环境配置
            environmentsElement(root.evalNode("environments"));
            //获取数据库厂商的标识
            databaseIdProviderElement(root.evalNode("databaseIdProvider"));
            //加载typeHandler
            typeHandlerElement(root.evalNode("typeHandlers"));
            //加载Mapper
            mapperElement(root.evalNode("mappers"));
        } catch (Exception e) {
            throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
        }
    }
```

## 总结
本篇文章讲述一下之后文章的一个路线，接下来会深入解析如何去加载，并且基础组件是怎么规划的，后续都会一一讲述

## 参考链接
- [Mybatis官网](https://mybatis.org/mybatis-3/zh/configuration.html)
- [mybatis源码解析3---XMLConfigBuilder解析](https://www.cnblogs.com/jackion5/p/9480393.html)
