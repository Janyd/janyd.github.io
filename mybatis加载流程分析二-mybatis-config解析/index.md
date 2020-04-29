# Mybatis加载流程分析（二）- mybatis-config解析


## Mybatis mybati-config.xml解析
从上一篇文章我们知道XMLConfigBuilder是对`mybatis-config.xml`进行解析，`XMLConfigBuilder`还有个抽象基类`BaseBuilder`，其中没有抽象方法仅仅抽取了公共方法。
{{< mermaid >}}
classDiagram
    class BaseBuilder
    <<abstract>> BaseBuilder
    class XMLConfigBuilder
    class XMLMapperBuilder
    class XMLStatementBuilder
    class XMLSqlSourceBuilder
    class XMLScriptBuilder
    class MapperBuilderAssistant
    BaseBuilder <|-- XMLConfigBuilder
    BaseBuilder <|-- XMLMapperBuilder
    BaseBuilder <|-- XMLStatementBuilder
    BaseBuilder <|-- XMLSqlSourceBuilder
    BaseBuilder <|-- XMLScriptBuilder
    BaseBuilder <|-- MapperBuilderAssistant
{{< /mermaid >}}
从以上类图看出`BaseBuilder`还有其他子类，有些从名称可以看出其作用，后续我们也会讲解。  

我们先来看`XMLConfigBuilder`类的结构：
### `XMLConfigBuilder`解析器
`XMLConfigBuilder`有六个构造方法，但最终调用的也只有一个构造方法
```Java
//是否已解析
private boolean parsed;
//Xpath解析器
private final XPathParser parser;
//环境标识
private String environment;
//反射器工厂
private final ReflectorFactory localReflectorFactory = new DefaultReflectorFactory();

public XMLConfigBuilder(Reader reader) {
        this(reader, null, null);
}

public XMLConfigBuilder(Reader reader, String environment) {
    this(reader, environment, null);
}

public XMLConfigBuilder(Reader reader, String environment, Properties props) {
    this(new XPathParser(reader, true, props, new XMLMapperEntityResolver()), environment, props);
}

public XMLConfigBuilder(InputStream inputStream) {
    this(inputStream, null, null);
}

public XMLConfigBuilder(InputStream inputStream, String environment) {
    this(inputStream, environment, null);
}

public XMLConfigBuilder(InputStream inputStream, String environment, Properties props) {
    this(new XPathParser(inputStream, true, props, new XMLMapperEntityResolver()), environment, props);
}
private XMLConfigBuilder(XPathParser parser, String environment, Properties props) {
    //1.调用父类构造方法
    super(new Configuration());
    ErrorContext.instance().resource("SQL Mapper Configuration");
    this.configuration.setVariables(props);
    this.parsed = false;
    this.environment = environment;
    this.parser = parser;
}
```
前面五个构造方法都是需要传入配置文件输入流，最终由`XPathParser`来读取，最后一个构造方法：
1. 调用父类构造方法，创建`Configuration`
2. 设置错误上下文的当前位置信息
3. 是否已解析设置为false
4. 环境标识
5. 解析器

我们再看看父类的构造函数：
```Java
protected final Configuration configuration;
protected final TypeAliasRegistry typeAliasRegistry;
protected final TypeHandlerRegistry typeHandlerRegistry;

public BaseBuilder(Configuration configuration) {
    this.configuration = configuration;
    this.typeAliasRegistry = this.configuration.getTypeAliasRegistry();
    this.typeHandlerRegistry = this.configuration.getTypeHandlerRegistry();
}
```
就是从`Configuration`取出别名注册类型与类型处理注册类方便在Builder类们所使用（这两个类在`Configuration`创建过程就初始化了，详情可看`Configuration`）

### `XPathParser`解析器  
该解析器位于`org.apache.ibatis.parsing`包中，里面包含了：  
![](https://cdn.jsdelivr.net/gh/Janyd/blog-images/imgs/20200425180512.png)  
`XPathParser`解析器是基于java提供的API来进行封装提供给Mybatis进行对XML的解析
#### `XPathParser`构造方法
```Java
/**
 * XML的Document内容
 */
private final Document document;


private boolean validation;

/**
 * XML 实体解析器，加载DTD或XSD文件
 */
private EntityResolver entityResolver;

/**
 * 加载变量，例如xml里有${}占位符时，需要通过额外的文件去加载变量值，动态替换属性值
 */
private Properties variables;

/**
 * Java Xpath 用于查询XML节点信息等等
 */
private XPath xpath;

/**
 * 公共构造函数
 *
 * @param validation     是否校验XML
 * @param variables      动态配置属性值
 * @param entityResolver XML实体解析器
*/
private void commonConstructor(boolean validation, Properties variables, EntityResolver entityResolver) {
    this.validation = validation;
    this.entityResolver = entityResolver;
    this.variables = variables;
    XPathFactory factory = XPathFactory.newInstance();
    this.xpath = factory.newXPath();
}

private Document createDocument(InputSource inputSource) {
    // important: this must only be called AFTER common constructor
    try {
        DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
        factory.setFeature(XMLConstants.FEATURE_SECURE_PROCESSING, true);
        //是否校验XML
        factory.setValidating(validation);

        factory.setNamespaceAware(false);
        factory.setIgnoringComments(true);
        factory.setIgnoringElementContentWhitespace(false);
        factory.setCoalescing(false);
        factory.setExpandEntityReferences(true);

        DocumentBuilder builder = factory.newDocumentBuilder();
        builder.setEntityResolver(entityResolver);

        //错误处理器
        builder.setErrorHandler(new ErrorHandler() {
            @Override
            public void error(SAXParseException exception) throws SAXException {
                throw exception;
            }

            @Override
            public void fatalError(SAXParseException exception) throws SAXException {
                throw exception;
            }

            @Override
            public void warning(SAXParseException exception) throws SAXException {
                // NOP
            }
        });
        return builder.parse(inputSource);
    } catch (Exception e) {
        throw new BuilderException("Error creating document instance.  Cause: " + e, e);
    }
}
```
`XPathParser`构造方法有很多，但基本都比较类似，都会调用`commonConstructor`与`createDocument`两个方法，属性的作用如下：
- `document`也就是XML的内容
- `validation`是否验证XML的正确性
- `entityResolver`XML的实体解析器，用于加载XML的DTD或XSD文件
- `variables`动态变量
- `xpath` 用于查询XML节点信息  

`createDocument`方法时为了创建`Document`，`Document`是`org.w3c.dom`包是java提供的。

其他的方法都是`eval*`的方法族，用于根据xpath表达式获取节点上的数据，我们重点来看`evaluate`方法：
```Java
/**
 * 解析xpath表达式
 *
 * @param expression 表达式
 * @param root       节点对象，从哪个节点开始
 * @param returnType 返回的类型
 * @return 数据
 */
private Object evaluate(String expression, Object root, QName returnType) {
    try {
        //利用xpath解析表达式
        return xpath.evaluate(expression, root, returnType);
    } catch (Exception e) {
        throw new BuilderException("Error evaluating XPath.  Cause: " + e, e);
    }
}
```
其他方法都是基于该方法来进行解析完成，但有部分是解析出节点对象`XNode`  
```Java
public XNode evalNode(Object root, String expression) {
    Node node = (Node) evaluate(expression, root, XPathConstants.NODE);
    if (node == null) {
        return null;
    }

    //封装成 @see org.apache.ibatis.parsing.XNode 主要是为了能够解析${xx} 动态值
    return new XNode(this, node, variables);
}
public List<XNode> evalNodes(Object root, String expression) {
        List<XNode> xnodes = new ArrayList<>();
        NodeList nodes = (NodeList) evaluate(expression, root, XPathConstants.NODESET);
        for (int i = 0; i < nodes.getLength(); i++) {
            xnodes.add(new XNode(this, nodes.item(i), variables));
        }
        return xnodes;
    }
```
`XNode`是mybatis对`org.w3c.dom.Node`的一个封装，增强节点对象的解析与动态变量的替换。
从上篇文章我们知道`XMLConfigBuilder#parseConfiguration`方法的参数就是`XNode`，这个这节点就是`mybatis-config.xml`的根节点`configuration`。  

### `XNode`解析动态配置
在`mybatis-config.xml`可以使用${}占位符来代表该变量是动态的，这个变量是从properties读取或在配置文件中节点`<properties />`读取的动态变量，而这里如何做到的呢？该相关的逻辑都在`XNode`与`PropertyParser`中，`XNode`获取标签属性时，会经过动态变量进行替换再返回，例如：
```Java
private Properties parseAttributes(Node n) {
    Properties attributes = new Properties();
    NamedNodeMap attributeNodes = n.getAttributes();
    if (attributeNodes != null) {
        for (int i = 0; i < attributeNodes.getLength(); i++) {
            Node attribute = attributeNodes.item(i);
            //此处调用PropertyParser#parse
            String value = PropertyParser.parse(attribute.getNodeValue(), variables);
            attributes.put(attribute.getNodeName(), value);
        }
    }
    return attributes;
}

//PropertyParser.java类
public static String parse(String string, Properties variables) {
    //变量处理器
    VariableTokenHandler handler = new VariableTokenHandler(variables);

    //通用解析器
    GenericTokenParser parser = new GenericTokenParser("${", "}", handler);
    //解析执行
    return parser.parse(string);
}
```
从代码可以看出使用了`VariableTokenHandler`与`GenericTokenParser`来进行解析，进一步查看`VariableTokenHandler`是`PropertyParser`内部类，并实现了接口`TokenHandler`  
```Java
private static class VariableTokenHandler implements TokenHandler {

    //动态变量
    private final Properties variables;
    //是否开启默认值
    private final boolean enableDefaultValue;
    //默认值的分隔符
    private final String defaultValueSeparator;

    private VariableTokenHandler(Properties variables) {
        this.variables = variables;
        this.enableDefaultValue = Boolean.parseBoolean(getPropertyValue(KEY_ENABLE_DEFAULT_VALUE, ENABLE_DEFAULT_VALUE));
        this.defaultValueSeparator = getPropertyValue(KEY_DEFAULT_VALUE_SEPARATOR, DEFAULT_VALUE_SEPARATOR);
    }

    private String getPropertyValue(String key, String defaultValue) {
        return (variables == null) ? defaultValue : variables.getProperty(key, defaultValue);
    }

    @Override
    public String handleToken(String content) {
        if (variables != null) {
            String key = content;
            if (enableDefaultValue) {
                //如果开启默认值，解析content里有没有 冒号(:)，冒号前是动态变量的key，冒号后面是默认值，如果key不存在variables里，就会取默认值
                final int separatorIndex = content.indexOf(defaultValueSeparator);
                String defaultValue = null;
                if (separatorIndex >= 0) {
                    key = content.substring(0, separatorIndex);
                    defaultValue = content.substring(separatorIndex + defaultValueSeparator.length());
                }
                if (defaultValue != null) {
                    return variables.getProperty(key, defaultValue);
                }
            }
            if (variables.containsKey(key)) {
                return variables.getProperty(key);
            }
        }
        return "${" + content + "}";
    }
}
```
从代码可看出其实就是处理content，并返回`variables`中的变量值，只是这里还含有默认值的处理，现在大多数都会采取这样的默认规则${key:defaultValue}  

`GenericTokenParser`是通用解析器，怎么通用呢？源码如下：
```Java
/**
 * 通用token解析器
 *
 * @author Clinton Begin
 */
public class GenericTokenParser {

    /**
     * 开始字符串
     */
    private final String openToken;

    /**
     * 结束字符串
     */
    private final String closeToken;

    /**
     * token处理器
     */
    private final TokenHandler handler;

    public GenericTokenParser(String openToken, String closeToken, TokenHandler handler) {
        this.openToken = openToken;
        this.closeToken = closeToken;
        this.handler = handler;
    }

    public String parse(String text) {
        if (text == null || text.isEmpty()) {
            return "";
        }
        // search open token
        int start = text.indexOf(openToken);
        if (start == -1) {
            return text;
        }
        char[] src = text.toCharArray();
        int offset = 0;
        final StringBuilder builder = new StringBuilder();
        StringBuilder expression = null;
        while (start > -1) {
            if (start > 0 && src[start - 1] == '\\') {
                // this open token is escaped. remove the backslash and continue.
                //由于openToken前是转义字符\,需要忽略
                builder.append(src, offset, start - offset - 1).append(openToken);
                //重新定位偏移位
                offset = start + openToken.length();
            } else {
                // found open token. let's search close token.
                if (expression == null) {
                    expression = new StringBuilder();
                } else {
                    expression.setLength(0);
                }
                builder.append(src, offset, start - offset);
                offset = start + openToken.length();
                int end = text.indexOf(closeToken, offset);
                while (end > -1) {
                    if (end > offset && src[end - 1] == '\\') {
                        // this close token is escaped. remove the backslash and continue.
                        expression.append(src, offset, end - offset - 1).append(closeToken);
                        offset = end + closeToken.length();
                        end = text.indexOf(closeToken, offset);
                    } else {
                        expression.append(src, offset, end - offset);
                        break;
                    }
                }
                if (end == -1) {
                    // close token was not found.
                    builder.append(src, start, src.length - start);
                    offset = src.length;
                } else {
                    builder.append(handler.handleToken(expression.toString()));
                    offset = end + closeToken.length();
                }
            }
            start = text.indexOf(openToken, offset);
        }
        if (offset < src.length) {
            builder.append(src, offset, src.length - offset);
        }
        return builder.toString();
    }
}
```
`GenericTokenParser`的三个属性显而易见，就是占位符的开始关键字符串`openToken`、结束关键字符串`closeToken`与占位符的处理器，这里的概念有点类似策略模式，定义了占位符处理器接口，解析器在一段字符串里找到占位符就交给处理器处理并替换占位符。而在这里只是处理了${}，在后续的解析中都会遇到该通用解析器

## 总结
本篇文章讲述了`XMLConfigBuilder`如何自己封装的解析器去解析`mybatis-config.xml`，由基于Java提供对XML解析的API之上封装出来的`XPathParser`解析器与增强`XNode`节点，并且了解了如何解析配置文件里${}动态配置的替换，
部分相关的源码解析地址：[https://github.com/Janyd/mybatis-3](https://github.com/Janyd/mybatis-3)



## 参考链接
- [MyBatis（三） xml文件解析流程 动态SQL解析](https://www.jianshu.com/p/c358a80155bc)
