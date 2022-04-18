# Spring Extensions

## Spring自定义标签

### 作用

在传统的Spring基于XML的开发流程中，自定义标签是Spring为开发人员编写扩展组件时使用的，实际上除了`Spring-core`和`Spring-beans`外所有的标签，其他标签都是通过自定义标签扩展实现的。Spring允许你通过自定义的XML标签进行Bean的解析和BeanDefinition的解析工作。

### 实现方式

- 在`resources/META-INF`路径下编写一个XML Schema的`.xsd`文件，来描述你的自定义标签

### 使用场景

- Apache Dubbo的`<dubbo:>`标签
- 自定义组件，例如数据源，我们可以通过`<bean>`标签初始化数据源Bean，也可以自定义一个`<datasource>`标签，完成Bean的定义并注册到容器中

## Spring Boot扩展点

**参考**

> 聊聊Spring中的那些扩展机制：http://blog.itpub.net/31555607/viewspace-2214762/
>
> 详解Spring Framework提供的扩展点：ApplicationContextInitializer应用上下文初始化器，以及它在SpringBoot中的应用: https://cloud.tencent.com/developer/article/1497696