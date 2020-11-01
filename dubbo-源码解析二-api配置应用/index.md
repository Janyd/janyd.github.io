# Dubbo 源码解析（二）  API配置（应用）


## AbstractConfig
`AbstractConfig`是除了`ArgumentConfig`外所有配置类的父类，提供了大部分通用的方法与属性。
### 属性
```java
//配置类后缀，在获取相关tag时，需要去除后缀
private static final String[] SUFFIXES = new String[]{"Config", "Bean", "ConfigBase"};

//配置编号，比如 ProviderConfig 编号就是provider 与 XML 对应<dubbo:provider>
protected String id;
//配置信息前缀，例如 dubbo.provider.
protected String prefix;
```

### 静态方法
```java
//将当前配置类转成小写中划线，作为配置类的 tag 名称，例如
public static String getTagName(Class<?> cls) {
    String tag = cls.getSimpleName();
    for (String suffix : SUFFIXES) {
        if (tag.endsWith(suffix)) {
            tag = tag.substring(0, tag.length() - suffix.length());
            break;
        }
    }
    return StringUtils.camelToSplitName(tag, "-");
}

/**
 * 将config类中配置添加到 parameters
 */
public static void appendParameters(Map<String, String> parameters, Object config) {
    appendParameters(parameters, config, null);
}

public static void appendParameters(Map<String, String> parameters, Object config, String prefix) {
    if (config == null) {
        return;
    }

    //获取config所有方法
    Method[] methods = config.getClass().getMethods();
    for (Method method : methods) {
        try {
            String name = method.getName();
            //获取getXX方法判定是需要添加到 parameters
            if (MethodUtils.isGetter(method)) {
                Parameter parameter = method.getAnnotation(Parameter.class);
                if (method.getReturnType() == Object.class || parameter != null && parameter.excluded()) {
                    //含有Parameter注解且excluded = true 或 返回Object
                    continue;
                }
                String key;
                //获取key，如注解上有设置key内容则获取注解中内容
                if (parameter != null && parameter.key().length() > 0) {
                    key = parameter.key();
                } else {
                    //否则从getXXX 字段名称
                    key = calculatePropertyFromGetter(name);
                }
                //获取该方法获取配置项值
                Object value = method.invoke(config);
                String str = String.valueOf(value).trim();
                if (value != null && str.length() > 0) {
                    if (parameter != null && parameter.escaped()) {
                        //需要将配置项内容编码
                        str = URL.encode(str);
                    }
                    if (parameter != null && parameter.append()) {
                        //追加配置项内容
                        String pre = parameters.get(key);
                        if (pre != null && pre.length() > 0) {
                            str = pre + "," + str;
                        }
                    }
                    if (prefix != null && prefix.length() > 0) {
                        //如有前缀需要加前缀
                        key = prefix + "." + key;
                    }
                    parameters.put(key, str);
                } else if (parameter != null && parameter.required()) {
                    throw new IllegalStateException(config.getClass().getSimpleName() + "." + key + " == null");
                }
            } else if (isParametersGetter(method)) { //判断是否存在 getParameters
                
                Map<String, String> map = (Map<String, String>) method.invoke(config, new Object[0]);
                parameters.putAll(convert(map, prefix));
            }
        } catch (Exception e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
    }
}

//从getter方法提取字段
private static String calculatePropertyFromGetter(String name) {
    int i = name.startsWith("get") ? 3 : 2;
    return StringUtils.camelToSplitName(name.substring(i, i + 1).toLowerCase() + name.substring(i + 1), ".");
}

/**
 * 将注解上配置信息设置到当前Config
 */
protected void appendAnnotation(Class<?> annotationClass, Object annotation) {
    //从注解获取相关配置名称
    Method[] methods = annotationClass.getMethods();
    for (Method method : methods) {
        if (method.getDeclaringClass() != Object.class
                && method.getReturnType() != void.class
                && method.getParameterTypes().length == 0
                && Modifier.isPublic(method.getModifiers())
                && !Modifier.isStatic(method.getModifiers())) {
            try {
                //获取配置项名称
                String property = method.getName();
                if ("interfaceClass".equals(property) || "interfaceName".equals(property)) {
                    property = "interface";
                }
                //将配置项转setter名称
                String setter = "set" + property.substring(0, 1).toUpperCase() + property.substring(1);
                //拿到配置项值
                Object value = method.invoke(annotation);
                if (value != null && !value.equals(method.getDefaultValue())) {
                    Class<?> parameterType = ReflectUtils.getBoxedClass(method.getReturnType());
                    //特殊配置项名称做相关处理
                    if ("filter".equals(property) || "listener".equals(property)) {
                        parameterType = String.class;
                        value = StringUtils.join((String[]) value, ",");
                    } else if ("parameters".equals(property)) {
                        parameterType = Map.class;
                        value = CollectionUtils.toStringMap((String[]) value);
                    }
                    try {
                        //利用反射拿到当前配置类的setter方法，并且将配置项值设置到当前类中，如果不存则会抛异常，在catch中进行忽略
                        Method setterMethod = getClass().getMethod(setter, parameterType);
                        setterMethod.invoke(this, value);
                    } catch (NoSuchMethodException e) {
                        // ignore
                    }
                }
            } catch (Throwable e) {
                logger.error(e.getMessage(), e);
            }
        }
    }
}

/**
 * 从环境变量中取出新的配置项，将配置类配置项进行刷新
 */
public void refresh() {
    Environment env = ApplicationModel.getEnvironment();
    try {
        CompositeConfiguration compositeConfiguration = env.getConfiguration(getPrefix(), getId());
        Configuration config = new ConfigConfigurationAdapter(this);
        if (env.isConfigCenterFirst()) {
            // The sequence would be: SystemConfiguration -> AppExternalConfiguration -> ExternalConfiguration -> AbstractConfig -> PropertiesConfiguration
            compositeConfiguration.addConfiguration(4, config);
        } else {
            // The sequence would be: SystemConfiguration -> AbstractConfig -> AppExternalConfiguration -> ExternalConfiguration -> PropertiesConfiguration
            compositeConfiguration.addConfiguration(2, config);
        }

        // loop methods, get override value and set the new value back to method
        Method[] methods = getClass().getMethods();
        for (Method method : methods) {
            if (MethodUtils.isSetter(method)) {
                try {
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

## ApplicationConfig
`org.apache.dubbo.config.ApplicationConfig`，应用配置。相关配置项解释，参见[《Dubbo指南 -- dubbo:application》](http://dubbo.apache.org/zh-cn/docs/2.7/user/references/xml/dubbo-application/)

## MonitorConfig
`org.apache.dubbo.config.MonitorConfig`，监控配置。相关配置项参见[《Dubbo指南 -- dubbo:monitor》](http://dubbo.apache.org/zh-cn/docs/2.7/user/references/xml/dubbo-monitor)

## ModuleConfig
`org.apache.dubbo.config.ModuleConfig`，模块配置。相关配置项参见[《Dubbo指南 -- dubbo:module》](http://dubbo.apache.org/zh-cn/docs/2.7/user/references/xml/dubbo-module)

## RegistryConfig
`org.apache.dubbo.config.RegistryConfig`，注册中心配置。相关配置项参见[《Dubbo指南 -- dubbo:registry》](http://dubbo.apache.org/zh-cn/docs/2.7/user/references/xml/dubbo-registry)

## ArgumentConfig
`org.apache.dubbo.config.ArgumentConfig`，方法参数配置。相关配置项参见[《Dubbo指南 -- dubbo:argument》](http://dubbo.apache.org/zh-cn/docs/2.7/user/references/xml/dubbo-argument)

## 参考链接
* [Dubbo官网](http://dubbo.apache.org/)
