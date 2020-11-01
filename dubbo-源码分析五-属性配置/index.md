# Dubbo 源码分析（五）    属性配置


## 概述
> 如果你的应用足够简单，例如，不需要多注册中心或多协议，并且需要在spring容器中共享配置，那么，我们可以直接使用 dubbo.properties作为默认配置。 Dubbo可以自动加载classpath根目录下的dubbo.properties，但是你同样可以使用JVM参数来指定路径：-Ddubbo.properties.file=xxx.properties。来自官方文档 [属性配置](http://dubbo.apache.org/zh-cn/docs/2.7/user/configuration/properties/)

从以上可以总结，使用属性配置有以下特点：
* 不需要多注册中心
* 外部化配置

## ConfigUtils
`org.apache.dubbo.common.utils.ConfigUtils`在此类中，`getProperties()`方法就进行对`dubbo.properties`配置文件进行读取。并且由`org.apache.dubbo.common.config.PropertiesConfiguration`来对外提供或者配置。
## AbstractConfig#refresh
各个配置类型获取入口在`AbstractConfig#refresh`中读取到设置到当前配置类中。一下便是`AbstractConfig#refresh`
```java
public void refresh() {
    //获取环境对象
    Environment env = ApplicationModel.getEnvironment();
    try {
        //根据prefix 与 id 获取当前组合配置信息
        CompositeConfiguration compositeConfiguration = env.getConfiguration(getPrefix(), getId());
        //并将当前封装成Configuration，以便复用部分配置。ConfigConfigurationAdapter适配器
        Configuration config = new ConfigConfigurationAdapter(this);
        //并且将该配置信息放入组合配置中，需要根据优先级放入
        if (env.isConfigCenterFirst()) {
            // The sequence would be: SystemConfiguration -> AppExternalConfiguration -> ExternalConfiguration -> AbstractConfig -> PropertiesConfiguration
            compositeConfiguration.addConfiguration(4, config);
        } else {
            // The sequence would be: SystemConfiguration -> AbstractConfig -> AppExternalConfiguration -> ExternalConfiguration -> PropertiesConfiguration
            compositeConfiguration.addConfiguration(2, config);
        }

        // loop methods, get override value and set the new value back to method
        //循环当前配置类方法，根据属性setter设置到当前配置类中
        Method[] methods = getClass().getMethods();
        for (Method method : methods) {
            if (MethodUtils.isSetter(method)) {
                try {
                    //此处就是读取配置值
                    String value = StringUtils.trim(compositeConfiguration.getString(extractPropertyName(getClass(), method)));
                    // isTypeMatch() is called to avoid duplicate and incorrect update, for example, we have two 'setGeneric' methods in ReferenceConfig.
                    if (StringUtils.isNotEmpty(value) && ClassUtils.isTypeMatch(method.getParameterTypes()[0], value)) {
                        method.invoke(this, ClassUtils.convertPrimitive(method.getParameterTypes()[0], value));
                    }
                } catch (NoSuchMethodException e) {
                    logger.info("Failed to override the property " + method.getName() + " in " +
                            this.getClass().getSimpleName() +
                            ", please make sure every property has getter/setter method provided.");
                }
            } else if (isParametersSetter(method)) {
                String value = StringUtils.trim(compositeConfiguration.getString(extractPropertyName(getClass(), method)));
                if (StringUtils.isNotEmpty(value)) {
                    Map<String, String> map = invokeGetParameters(getClass(), this);
                    map = map == null ? new HashMap<>() : map;
                    map.putAll(convert(StringUtils.parseParameters(value), ""));
                    invokeSetParameters(getClass(), this, map);
                }
            }
        }
    } catch (Exception e) {
        logger.error("Failed to override ", e);
    }
}
```
组合配置中包含有多个配置指示着，`Dubbo`可从多个途径读取到配置信息。下图就是多个途径读取优先级，优先级高会重写优先极低配置。
>优先级从高到低：  
JVM -D参数，当你部署或者启动应用时，它可以轻易地重写配置，比如，改变dubbo协议端口；  
XML, XML中的当前配置会重写dubbo.properties中的；  
Properties，默认配置，仅仅作用于以上两者没有配置时。  

![](https://cdn.jsdelivr.net/gh/Janyd/blog-images/imgs/20201101190905.png)


## 参考链接

