# Mybatis加载流程分析(三) - 加载mybatis-config.xml内容


## 动态配置加载propertiesElement
Mybatis中的动态配置指的是上篇${}占位符替换，而这些变量来自于`mybatis-config.xml`中节点`<properties />`，或者从`XMLConfigBuilder`构造函数传入`prop`，`mybatis-config.xml`节点`<properties />`也有两种方式：
```xml
<properties resource="org/apache/ibatis/databases/blog/blog-derby.properties"> //properties文件地址
    <property name="username" value="username"/> 在properties节点里设置属性值
    <property name="password" value="password"/>
</properties>
```
而三种种方式优先级是：
{{< mermaid >}}
graph TD
构造函数prop --> properties文件
properties文件 --> property节点
{{< /mermaid >}}
优先级越低就有可能相同变量名就会被优先级越高的覆盖，我们可以看`XMLConfigBuilder#propertiesElement`源码  
```Java
private void propertiesElement(XNode context) throws Exception {
    if (context != null) {
        //先读取子节点变量
        Properties defaults = context.getChildrenAsProperties();
        String resource = context.getStringAttribute("resource");
        String url = context.getStringAttribute("url");
        if (resource != null && url != null) {

            throw new BuilderException("The properties element cannot specify both a URL and a resource based property file reference.  Please specify one or the other.");
        }
        //后加载properties文件，并且添加到default，如有相同键名就会被覆盖
        if (resource != null) {
            defaults.putAll(Resources.getResourceAsProperties(resource));
        } else if (url != null) {
            defaults.putAll(Resources.getUrlAsProperties(url));
        }
        //取出先前构造函数传入的变量，然后添加进default，如有相同键名会将构造函数传入的变量覆盖于上面两个方式的变量
        Properties vars = configuration.getVariables();
        if (vars != null) {
            defaults.putAll(vars);
        }
        parser.setVariables(defaults);
        configuration.setVariables(defaults);
    }
}
```
这里还有个点就是`url` 或 `resource`不能同时配置，`resource`其实就是本地文件地址，而`url`统一资源定位符，这个可以指定一个网址只要他返回properties文件内容即可

## settings节点解析settingsAsProperties
> 
```Java
/**
    * 解析settings节点的配置信息，并且校验Configuration是否存在该字段
    *
    * @param context
    * @return
    */
private Properties settingsAsProperties(XNode context) {
    if (context == null) {
        return new Properties();
    }
    Properties props = context.getChildrenAsProperties();
    // Check that all settings are known to the configuration class
    MetaClass metaConfig = MetaClass.forClass(Configuration.class, localReflectorFactory);
    for (Object key : props.keySet()) {
        if (!metaConfig.hasSetter(String.valueOf(key))) {
            throw new BuilderException("The setting " + key + " is not known.  Make sure you spelled it correctly (case sensitive).");
        }
    }
    return props;
}
```
settings解析很简单，解析的内容主要是`Configuration`配置项，完整配置如下：
```xml
<settings>
  <setting name="cacheEnabled" value="true"/>
  <setting name="lazyLoadingEnabled" value="true"/>
  <setting name="multipleResultSetsEnabled" value="true"/>
  <setting name="useColumnLabel" value="true"/>
  <setting name="useGeneratedKeys" value="false"/>
  <setting name="autoMappingBehavior" value="PARTIAL"/>
  <setting name="autoMappingUnknownColumnBehavior" value="WARNING"/>
  <setting name="defaultExecutorType" value="SIMPLE"/>
  <setting name="defaultStatementTimeout" value="25"/>
  <setting name="defaultFetchSize" value="100"/>
  <setting name="safeRowBoundsEnabled" value="false"/>
  <setting name="mapUnderscoreToCamelCase" value="false"/>
  <setting name="localCacheScope" value="SESSION"/>
  <setting name="jdbcTypeForNull" value="OTHER"/>
  <setting name="lazyLoadTriggerMethods" value="equals,clone,hashCode,toString"/>
</settings>
```
具体含义可到[Mybatis官网查看](https://mybatis.org/mybatis-3/zh/configuration.html#settings)  

这里要说两个就是`VFS`与`Log`，`Log`比较容易理解其实就是Mybatis框架的日志打印，可由用户自行实现也可用Mybatis自带。`VFS`指的就是VFS（virtual File System）虚拟文件系统，主要用来加载容器内的各种资源，比如jar或class文件。Mybatis内部提供`DefaultVFS`与`JBoss6VFS`，`VFS`有两个抽象方法：
```Java
//是否生效
public abstract boolean isValid();
//递归列出某个文件夹下所有子资源的全路径
protected abstract List<String> list(URL url, String forPath) throws IOException;
```
`list`仅在`VFS#list(path)`所使用：
```Java
public List<String> list(String path) throws IOException {
    List<String> names = new ArrayList<>();
    for (URL url : getResources(path)) {
        names.addAll(list(url, path));
    }
    return names;
}
```
而该方法也只会`ResolverUtil<T>`所使用的，而`ResolverUtil<T>`的作用为了能够获取某些包名下的`.class`类文件。  
这几类属于`org.apache.ibatis.io`模块里，属于专门处理io。

## 加载类型别名typeAliasesElement
何为类型别名？直接看到`TypeAliasRegistry`：
```Java
public class TypeAliasRegistry {

    private final Map<String, Class<?>> typeAliases = new HashMap<>();

    //构造函数预注册
    public TypeAliasRegistry() {
        registerAlias("string", String.class);

        registerAlias("byte", Byte.class);
        registerAlias("long", Long.class);
        registerAlias("short", Short.class);
        registerAlias("int", Integer.class);
        registerAlias("integer", Integer.class);
        registerAlias("double", Double.class);
        registerAlias("float", Float.class);
        registerAlias("boolean", Boolean.class);

        registerAlias("byte[]", Byte[].class);
        registerAlias("long[]", Long[].class);
        registerAlias("short[]", Short[].class);
        registerAlias("int[]", Integer[].class);
        registerAlias("integer[]", Integer[].class);
        registerAlias("double[]", Double[].class);
        registerAlias("float[]", Float[].class);
        registerAlias("boolean[]", Boolean[].class);

        registerAlias("_byte", byte.class);
        registerAlias("_long", long.class);
        registerAlias("_short", short.class);
        registerAlias("_int", int.class);
        registerAlias("_integer", int.class);
        registerAlias("_double", double.class);
        registerAlias("_float", float.class);
        registerAlias("_boolean", boolean.class);

        registerAlias("_byte[]", byte[].class);
        registerAlias("_long[]", long[].class);
        registerAlias("_short[]", short[].class);
        registerAlias("_int[]", int[].class);
        registerAlias("_integer[]", int[].class);
        registerAlias("_double[]", double[].class);
        registerAlias("_float[]", float[].class);
        registerAlias("_boolean[]", boolean[].class);

        registerAlias("date", Date.class);
        registerAlias("decimal", BigDecimal.class);
        registerAlias("bigdecimal", BigDecimal.class);
        registerAlias("biginteger", BigInteger.class);
        registerAlias("object", Object.class);

        registerAlias("date[]", Date[].class);
        registerAlias("decimal[]", BigDecimal[].class);
        registerAlias("bigdecimal[]", BigDecimal[].class);
        registerAlias("biginteger[]", BigInteger[].class);
        registerAlias("object[]", Object[].class);

        registerAlias("map", Map.class);
        registerAlias("hashmap", HashMap.class);
        registerAlias("list", List.class);
        registerAlias("arraylist", ArrayList.class);
        registerAlias("collection", Collection.class);
        registerAlias("iterator", Iterator.class);

        registerAlias("ResultSet", ResultSet.class);
    }

    //先忽略其他方法
}
```
这个是一个别名类型，意思就是给出一个别名能够使用别名找到该类型，一般用在解析ResultMap中。而在`XMLConfigBuilder`加载的是用户自定义的别名类型或指定包，一般`package`会将整个`entity`都注册进别名类型中，代码如下：
```Java
/**
 * 加载别名节点配置
 *
 * @param parent
 */
private void typeAliasesElement(XNode parent) {
    if (parent != null) {
        for (XNode child : parent.getChildren()) {
            if ("package".equals(child.getName())) {
                String typeAliasPackage = child.getStringAttribute("name");
                configuration.getTypeAliasRegistry().registerAliases(typeAliasPackage);
            } else {
                String alias = child.getStringAttribute("alias");
                String type = child.getStringAttribute("type");
                try {
                    Class<?> clazz = Resources.classForName(type);
                    if (alias == null) {
                        typeAliasRegistry.registerAlias(clazz);
                    } else {
                        typeAliasRegistry.registerAlias(alias, clazz);
                    }
                } catch (ClassNotFoundException e) {
                    throw new BuilderException("Error registering typeAlias for '" + alias + "'. Cause: " + e, e);
                }
            }
        }
    }
}
```

## 加载插件pluginElement
Mybatis为用户提供了插件系统，能允许用户在映射语句执行过程进行拦截调用，更改Mybatis的默认行为例如修改SQL等等，以下是Mybatis允许用户使用插件进行拦截接口方法：
- org.apache.ibatis.executor.Executor(update、query、queryCursor、flushStatements、commit、rollback、close、isClosed)
- org.apache.ibatis.executor.statement.StatementHandler(prepare、batch、update、query)
- org.apache.ibatis.executor.parameter.ParameterHandler(getParameterObject、setParameters)
- org.apache.ibatis.executor.resultset.ResultSetHandler(handleResultSets、handleCursorResultSets、handleOutputParameters)
Mybatis提供了`org.apache.ibatis.plugin.Interceptor`接口来实现插件，但实现接口还不行，还需要加上注解`org.apache.ibatis.plugin.Intercepts`与`org/apache/ibatis/plugin/Signature`，`Intercepts`主要是包含多个`Signature`，重点在于`Signature`属性：
```Java
/**
 * The annotation that indicate the method signature.
 *
 * @see Intercepts
 * @author Clinton Begin
 */
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({})
public @interface Signature {
    /**
     * 指定四大金刚：Executor、StatementHandler、ParameterHandler、ResultSetHandler
     * Returns the java type.
     *
     * @return the java type
     */
    Class<?> type();

    /**
     * 指定上面接口中的方法
     * Returns the method name.
     *
     * @return the method name
     */
    String method();

    /**
     * 指定上面方法的参数类型，并必须
     * Returns java types for method argument.
     * @return java types for method argument
     */
    Class<?>[] args();
}
```
现在最常见的插件就是分页插件，例如`PageHelper`([Github坐标](https://github.com/pagehelper/Mybatis-PageHelper))。

## 加载对象工厂
在Mybatis里对象工厂的作用就是实例化，并且支持有构造函数，同时支持集合类(List、Map等等)的创建，主要是Mybatis中有结果集映射，需要返回相应的类型，有时候多个结果时那肯定是需要集合的，所以需要一个对象工厂来帮忙实例化并设置值，Mybatis已经提供一个默认的`DefaultObjectFactory`，加载过程如下：
```Java
/**
 * 加载自定义对象工厂实现类
 * <objectFactory type="org.mybatis.example.ExampleObjectFactory">
 * <property name="someProperty" value="100"/>
 * </objectFactory>
 *
 * @param context
 * @throws Exception
 */
private void objectFactoryElement(XNode context) throws Exception {
    if (context != null) {
        String type = context.getStringAttribute("type");
        Properties properties = context.getChildrenAsProperties();
        ObjectFactory factory = (ObjectFactory) resolveClass(type).getDeclaredConstructor().newInstance();
        factory.setProperties(properties);
        configuration.setObjectFactory(factory);
    }
}
```

## 加载对象包装工厂objectWrapperFactoryElement
此处加载用户定义的`ObjectWrapperFactory`的实现类，系统也提供了`DefaultObjectWrapperFactory`：
```Java
/**
 * @author Clinton Begin
 */
public class DefaultObjectWrapperFactory implements ObjectWrapperFactory {

    @Override
    public boolean hasWrapperFor(Object object) {
        return false;
    }

    @Override
    public ObjectWrapper getWrapperFor(MetaObject metaObject, Object object) {
        throw new ReflectionException("The DefaultObjectWrapperFactory should never be called to provide an ObjectWrapper.");
    }

}
```
从代码看来是个不实现任何内容的空类，Mybaits默认不实现该类，而查看方法引用只有在`org.apache.ibatis.reflection.MetaObject`的构造函数引用：
```Java
public class MetaObject {

    //原对象
    private final Object originalObject;
    //对象封装
    private final ObjectWrapper objectWrapper;
    //对象工厂
    private final ObjectFactory objectFactory;
    //对象工厂
    private final ObjectWrapperFactory objectWrapperFactory;
    //反射器工厂
    private final ReflectorFactory reflectorFactory;

    private MetaObject(Object object, ObjectFactory objectFactory, ObjectWrapperFactory objectWrapperFactory, ReflectorFactory reflectorFactory) {
        this.originalObject = object;
        this.objectFactory = objectFactory;
        this.objectWrapperFactory = objectWrapperFactory;
        this.reflectorFactory = reflectorFactory;

        if (object instanceof ObjectWrapper) {

            this.objectWrapper = (ObjectWrapper) object;
        } else if (objectWrapperFactory.hasWrapperFor(object)) {
            //此处使用到了对象包装工厂
            this.objectWrapper = objectWrapperFactory.getWrapperFor(this, object);
        } else if (object instanceof Map) {
            this.objectWrapper = new MapWrapper(this, (Map) object);
        } else if (object instanceof Collection) {
            this.objectWrapper = new CollectionWrapper(this, (Collection) object);
        } else {
            this.objectWrapper = new BeanWrapper(this, object);
        }
    }

    //省略
}
```
而`org.apache.ibatis.reflection.wrapper.ObjectWrapper`代码如下：
```Java
/**
 * 对象封装，基于MetaClass
 *
 * @author Clinton Begin
 */
public interface ObjectWrapper {

    Object get(PropertyTokenizer prop);

    void set(PropertyTokenizer prop, Object value);

    String findProperty(String name, boolean useCamelCaseMapping);

    String[] getGetterNames();

    String[] getSetterNames();

    Class<?> getSetterType(String name);

    Class<?> getGetterType(String name);

    boolean hasSetter(String name);

    boolean hasGetter(String name);

    MetaObject instantiatePropertyValue(String name, PropertyTokenizer prop, ObjectFactory objectFactory);

    boolean isCollection();

    void add(Object element);

    <E> void addAll(List<E> element);

}
```
能够清晰的知道对一个对象封装，能够快速对对象getter与setter，`org.apache.ibatis.reflection.wrapper.ObjectWrapper`有`BeanWrapper`、`MapWrapper`、`CollectionWrapper`实现，当然你可以自己实现。

## 加载反射器工厂reflectorFactoryElement
```Java
private void reflectorFactoryElement(XNode context) throws Exception {
    if (context != null) {
        String type = context.getStringAttribute("type");
        ReflectorFactory factory = (ReflectorFactory) resolveClass(type).getDeclaredConstructor().newInstance();
        configuration.setReflectorFactory(factory);
    }
}
```
加载用户自定义的`ReflectorFactory`反射器工厂，如果没有Mybaits提供了默认实现类`DefaultReflectorFactory`，该接口的作用是能够获取得到`Reflector`：
```Java
public class Reflector {

    /**
     * 反射的Class类型
     */
    private final Class<?> type;

    /**
     * 可通过getter读取的字段数组
     */
    private final String[] readablePropertyNames;

    /**
     * 可通过setter写入的字段数组
     */
    private final String[] writablePropertyNames;

    /**
     * setter方法Method并封装在Invoker内，方便于执行setter方法，key ->字段名称 value -> set的 Invoker
     */
    private final Map<String, Invoker> setMethods = new HashMap<>();

    /**
     * getter方法Method方法封装在Invoker内，方便于执行getter方法，key->字段名称 value-> get的 Invoker
     */
    private final Map<String, Invoker> getMethods = new HashMap<>();

    /**
     * setter方法，入参类型
     */
    private final Map<String, Class<?>> setTypes = new HashMap<>();

    /**
     * getter方法，出参类型
     */
    private final Map<String, Class<?>> getTypes = new HashMap<>();

    /**
     * Class 默认构造函数 ，一般为无参数构造函数
     */
    private Constructor<?> defaultConstructor;

    /**
     * 字段大小写转换，大写字段名key -> 小写字段名value
     */
    private Map<String, String> caseInsensitivePropertyMap = new HashMap<>();

    public Reflector(Class<?> clazz) {
        //反射器处理的Class
        type = clazz;

        //添加默认构造函数Method
        addDefaultConstructor(clazz);

        //查找并添加getter方法
        addGetMethods(clazz);

        //查找并添加setter方法
        addSetMethods(clazz);

        //查找Class的字段
        addFields(clazz);

        //获取可通过getter字段名
        readablePropertyNames = getMethods.keySet().toArray(new String[0]);
        //获取可通过setter的字段名
        writablePropertyNames = setMethods.keySet().toArray(new String[0]);


        for (String propName : readablePropertyNames) {
            caseInsensitivePropertyMap.put(propName.toUpperCase(Locale.ENGLISH), propName);
        }
        for (String propName : writablePropertyNames) {
            caseInsensitivePropertyMap.put(propName.toUpperCase(Locale.ENGLISH), propName);
        }
    }
    //省略
}
```
从代码可看出是存储了`Class`的元信息，并且哪些属性是可getter、setter以及相应的方法，就算属性没有getter、setter也可以设置可访问性，强制给对象设置值。

## 加载环境
```Java
/**
 * 加载环境，用于不同环境不同配置
 *
 * @param context
 * @throws Exception
 */
private void environmentsElement(XNode context) throws Exception {
    if (context != null) {
        if (environment == null) {
            environment = context.getStringAttribute("default");
        }
        for (XNode child : context.getChildren()) {
            String id = child.getStringAttribute("id");
            if (isSpecifiedEnvironment(id)) {
                TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
                DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
                DataSource dataSource = dsFactory.getDataSource();
                Environment.Builder environmentBuilder = new Environment.Builder(id)
                    .transactionFactory(txFactory)
                    .dataSource(dataSource);
                configuration.setEnvironment(environmentBuilder.build());
            }
        }
    }
}
```
加载环境主要是要加载数据源信息与选择事务管理器实现类，这里的环境是可以配置多个的，然后选择一个默认，在加载过程就会选择默认那个环境进行初始化，配置如下：
```xml
<environments default="development">
  <environment id="development">
    <transactionManager type="JDBC">
      <property name="..." value="..."/>
    </transactionManager>
    <dataSource type="POOLED">
      <property name="driver" value="${driver}"/>
      <property name="url" value="${url}"/>
      <property name="username" value="${username}"/>
      <property name="password" value="${password}"/>
    </dataSource>
  </environment><environment id="test">
    <transactionManager type="JDBC">
      <property name="..." value="..."/>
    </transactionManager>
    <dataSource type="POOLED">
      <property name="driver" value="${driver}"/>
      <property name="url" value="${url}"/>
      <property name="username" value="${username}"/>
      <property name="password" value="${password}"/>
    </dataSource>
  </environment>
</environments>
```

## 加载类型映射typeHandlerElement
有时候我们遇到entity中有个属性是个复杂对象，对应着数据库应该一个字段，这个时候我们可以使用`TypeHandler`来进行映射，所以有了自定义类型映射：
```Java
/**
 * 加载用户自定义TypeHandler
 *
 * @param parent
 */
private void typeHandlerElement(XNode parent) {
    if (parent != null) {
        for (XNode child : parent.getChildren()) {
            if ("package".equals(child.getName())) {
                String typeHandlerPackage = child.getStringAttribute("name");
                typeHandlerRegistry.register(typeHandlerPackage);
            } else {
                String javaTypeName = child.getStringAttribute("javaType");
                String jdbcTypeName = child.getStringAttribute("jdbcType");
                String handlerTypeName = child.getStringAttribute("handler");
                Class<?> javaTypeClass = resolveClass(javaTypeName);
                JdbcType jdbcType = resolveJdbcType(jdbcTypeName);
                Class<?> typeHandlerClass = resolveClass(handlerTypeName);
                if (javaTypeClass != null) {
                    if (jdbcType == null) {
                        typeHandlerRegistry.register(javaTypeClass, typeHandlerClass);
                    } else {
                        typeHandlerRegistry.register(javaTypeClass, jdbcType, typeHandlerClass);
                    }
                } else {
                    typeHandlerRegistry.register(typeHandlerClass);
                }
            }
        }
    }
}
```
所有`TypeHandler`都需要注册到`TypeHandlerRegistry`，`TypeHandlerRegistry`内已经集成了很多基础的`TypeHandler`，有兴趣的可以看一下。

## 总结
本篇文章讲解了部分加载流程内容，讲解了动态配置、VFS资源读取、类型别名、插件、对象工厂、对象包装工厂、反射器工厂、环境配置、类型映射，当然加载流程还没完接下来的才是重要的部分`Mapper.xml`的解析，也是最复杂的一部分。

## 参考链接
- [MyBatis 3 | 配置](https://mybatis.org/mybatis-3/zh/configuration.html#settings)
- [加载自定义VFS loadCustomVfs](https://www.cnblogs.com/zhjh256/p/8512392.html#2.1.3-%E5%8A%A0%E8%BD%BD%E8%87%AA%E5%AE%9A%E4%B9%89vfs-loadcustomvfs)
- [在mybatis配置实体类的别名的两种常用配置方法](https://www.jianshu.com/p/a5ad61aa0973)
- [深入理解Mybatis插件开发](https://www.cnblogs.com/chenpi/p/10498921.html)
