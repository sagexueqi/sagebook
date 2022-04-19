# Spring Extensions

## 自定义标签

### 作用

> BeanDefinition阶段

在传统的Spring基于XML的开发流程中，自定义标签是Spring为开发人员编写扩展组件时使用的，实际上除了`Spring-core`和`Spring-beans`外所有的标签，其他标签都是通过自定义标签扩展实现的。Spring允许你通过自定义的XML标签进行Bean的解析和BeanDefinition的解析工作。

### 实现方式

- 在`resources/META-INF`路径下编写一个XML Schema的`.xsd`文件，来描述你的自定义标签
```xml
<?xml version="1.0" encoding="UTF-8"?>
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
           xmlns="http://lizard.com/schema/context"
           elementFormDefault="qualified"
           targetNamespace="http://lizard.com/schema/context">

    <!-- 定义一个lizard-context的标签 -->

    <!-- 定义标签里的一个place-holder元素 -->
    <xs:element name="place-holder">
        <xs:complexType>
            <!-- 元素中定义一个id属性 -->
            <xs:attribute name="id" use="optional" type="xs:string" default="propertyConfigurer"/>
        </xs:complexType>
    </xs:element>
</xs:schema>
```

- 在`resources/META-INF`路径下创建`spring.schemas`文件，用来指定引入标签url是指定的xsd文件路径
```properties
# 表示xml中引入http\://lizard.com/schema/context-1.0.0.xsd时指向/META-INF/lizard-context.xsd
http\://lizard.com/schema/context-1.0.0.xsd=/META-INF/lizard-context.xsd
```

- 定义`NamespaceHandlerSupport`，注册`BeanDefinitionParser`以支持标签中元素解析的处理逻辑；每一个标签对应一个`NamespaceHandlerSupport`,但是一个`NamespaceHandlerSupport`可以支持标签下多个元素的解析配置
```java
public class LizardContextNameSpaceHandler extends NamespaceHandlerSupport {
    @Override
    public void init() {
        // 针对每一个元素的解析处理，其实还是注册一个对应的BeanDefinitionParser
        // 这说明自定义标签的目的还是在于解析Bean定义
        super.registerBeanDefinitionParser("place-holder", new ContextPlaceHolderBeanDefinitionParser());
    }
}
```

- 定义`BeanDefinitionParser`，以支持具体元素的解析处理过程；基本上就是组装BeanDefinition并注册到Spring容器中的过程
```java
public class ContextPlaceHolderBeanDefinitionParser implements BeanDefinitionParser {
    @Override
    public BeanDefinition parse(Element element, ParserContext parserContext) {
        // 模拟将Demo Bean注册到容器中，一般来说，我们会在解析的过程中注册很多的BeanDefinition到容器中
        // 所以在parse方法中，我们一般都返回null即可

        String id = element.getAttribute("id");
        RootBeanDefinition demoBeanDefinition = new RootBeanDefinition(Demo.class);
        demoBeanDefinition.getPropertyValues().add("name", "sage");
        parserContext.getRegistry().registerBeanDefinition("demoBean", demoBeanDefinition);
        return null;
    }
}
```

- 在`resources/META-INF`路径下创建`spring.handlers`文件，以指定标签url对应的处理器类
```properties
# 将标签的url映射到对应的解析器
http\://lizard.com/schema/context=io.lizard.spring.extensions.namespace.LizardContextNameSpaceHandler
```

- 在spring配置文件中引入标签并使用
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:lizard-context="http://lizard.com/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://lizard.com/schema/context http://lizard.com/schema/context-1.0.0.xsd"
       default-lazy-init="false">

    <lizard-context:place-holder id="test"></lizard-context:place-holder>

</beans>
```

### 使用场景

- Apache Dubbo的`<dubbo:>`标签，通过自定义标签，将所有需要的Bean Definition注册到容器中；在Spring容器启动的过程中，完成Bean的创建
- 自定义组件，例如数据源，我们可以通过`<bean>`标签初始化数据源Bean，也可以自定义一个`<datasource>`标签，完成Bean的定义并注册到容器中；同时也可以在这一步，调用远程接口获取数据源连接信息等，直接组装到Bean的属性中

----

## BeanFactoryPostProcessor

### 作用

> BeanDefinition阶段

可以在Bean被实例化之前，对Bean的Definition进行修改。

### 实现方式

**一般用法**

实现BeanFactoryPostProcessor接口，重写postProcessBeanFactory方法，完成对BeanDefinition的修改；并将实现类注册到Spring容器中

```java
public class DemoBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory) throws BeansException {
        // 获取指定的BeanDefinition
        BeanDefinition demoDefinition = configurableListableBeanFactory.getBeanDefinition("demo");
        // 替换BeanDefinition中的属性值
        demoDefinition.getPropertyValues().add("name", "xueqi");

        // 获取BeanName列表，再获取BeanDefinition进行修改
        String[] beanNames = configurableListableBeanFactory.getBeanDefinitionNames();
        for (String beanName : beanNames) {
            BeanDefinition beanDefinition = configurableListableBeanFactory.getBeanDefinition(beanName);
            // 针对指定的Class进行处理
            if (beanDefinition.getBeanClassName().equals(Demo.class.getName())){
                beanDefinition.getPropertyValues().add("name", "xueqi-bean");
            }
        }
    }
}
```

**高级实现：PropertyPlaceholderConfigurer；对Definition中的`${}`占位符的值进行替换**

继承PropertyPlaceholderConfigurer类，重写`loadProperties`方法，实现占位符值的替换逻辑

```java
public class DemoPropertyPlaceholderConfigurer extends PropertyPlaceholderConfigurer {
    @Override
    protected void loadProperties(Properties props) throws IOException {
        // 父类装载过程不能忽略
        super.loadProperties(props);

        // 替换所有Bean属性value中的${}占位符的值
        props.put("key.xueqi", "xueqi-key");
    }
}
```

### 使用场景

- BeanFactoryPostProcessor会通过BeanFactory的getBean方法提前实例化，但是不会注入属性和初始化，所以无法注入Spring Bean
- 针对Bean的属性进行替换，通过配置中心等方式

----

## Aware

### 作用

> Bean初始化阶段 - 依赖注入后，初始化前

在实际开发中，我们可能需要在Bean中使用Spring容器本身的资源和功能。此时，Bean就需要实现对应的Aware接口，Spring容器在Bean的初始化阶段会将与之对应的功能对象通过回调的方式，传入对应的Spring容器资源对象供Bean进行使用。

Bean是无法感知到自己存在Spring容器之中，Aware的作用是为了让Bean感知到容器存在，并使用容器提供的功能。虽然破坏了低耦合和低侵入性，但是增加了使用的便利性

### 实现方式

实现相应的Aware接口

**常见的Aware**

- BeanNameAware: 获取该Class在容器中的Bean名称
- BeanFactoryAware：注入当前Bean的BeanFactory，从而调用容器的服务
- ApplicationContextAware：获取当前Bean的ApplicationContext，从而调用容器服务
> 一般来说常用的Aware是ApplicationContextAware，因为它集成了MessageSource接口、ApplicationEventPublisher接口和ResourceLoader接口，因此当Bean继承自ApplicationContextAware的时候就可以得到Spring容器的所有服务
- EnvironmentAware：将上下文中Enviroment注入进去，一般获取配置属性时可以使用

----

## BeanPostProcessor

----

## @PostConstruct

----

## @PreDestroy

----

## ApplicationContextInitializer

----

## AutoConfiguration

**参考**

> 聊聊Spring中的那些扩展机制：http://blog.itpub.net/31555607/viewspace-2214762/
>
> 详解Spring Framework提供的扩展点：ApplicationContextInitializer应用上下文初始化器，以及它在SpringBoot中的应用: https://cloud.tencent.com/developer/article/1497696
>
> @PostConstruct注解: https://www.jianshu.com/p/c52713a78252