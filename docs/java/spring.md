# Spring

## 基本概念

#### IOC: 控制反转
- Spring作为IOC容器，可以理解为一个非常大的Map
- Map中维持着对象名称和实例之间的映射关系

#### DI: 依赖注入
- IOC是一种思想，将实例和名称维护在一个大的容器中，将Bean与Bean之间的关系，交由容器维护
- DI是实现这种思想的手段，依赖由容器注入到使用方中

#### AOP: 面向切面编程
- 目标：提高代码复用，将通用的日志、拦截增强等操作进行抽象
- 通过**代理**方式实现AOP

**静态代理**
- 编译期实现增强，如`Aspect`

**动态代理**
- JDK动态代理
  - 基于接口实现。原因: 基于JDK动态代理生成的类，默认继承Proxy类，Java是不支持多继承
  - 通过 `ProxyGenerator.generateProxyClass` 生成动态代理类字节码, 通过native的findClass方法装载Class到Metaspace区
- CGLIB动态代理
  - 基于类实现。生成的动态代理类直接继承被代理类，完成增强环绕。所以，final方法是无法被代理的
  - 基于ASM改写字节码

-----

## Spring AOP

**在Bean实例化完成并注入属性后，初始化Bean之前。执行`AbstractAutoProxyCreator`的`postProcessBeforeInstantiation`方法，返回通过AOP增强的动态代理类对象**
- 即执行`BeanPostProcessor`的`before`方法
- 如果有Advised增强作用于当前bean，例如@Transactional、自定义AOP环绕等，则通过JDK或CGLIB动态代理生成代理类
- 生成的代理类继承Advised接口,表示被增强
- 否则直接创建类或接口实现类对象本身

> 为什么是执行`BeanPostProcessor`的`before`方法?
>
> 根据Bean的初始化流程来看，执行init方法前，Bean的属性应该是被动态代理完全的，总不能让初始化方法用一个还没有被增强的Bean
>
> 例如一个实际场景：bean初始化逻辑中要读取数据库，如果依赖的Bean还没有被增强，如何获取事务？如何获取JDBC Connection？

[AbstractAutoProxyCreator.java]
```java
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {

    // ...省略其他步骤...

    Object[] specificInterceptors = this.getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
    // 创建动态代理类对象
    Object proxy = this.createProxy(beanClass, beanName, specificInterceptors, targetSource);
    this.proxyTypes.put(cacheKey, proxy.getClass());

    return proxy;
}
```

**Spring AOP与Aspect的关系**
- Spring实现了Aspect的注解规范，例如@Around @Before等，并没有基于Aspect实现代理

**如何强制使用Cglib生成动态代理类**```target-proxy=true```
- 如果不使用该配置，默认根据类是否实现了接口，决定采用哪种方式实现动态代理

**参考:**
> 从动态代理实现到Spring AOP看这一篇就够了: https://mp.weixin.qq.com/s/EZ62B31Wxy47ISpiEKKVlg

----

## Spring Core

### 基本概念

- 实例化：创建Bean的原始对象，可以理解为只调用构造方法
- 依赖注入：根据BeanDefinition完成依赖Bean的实例化 + 注入 + 初始化(这是一个嵌套递归的过程，最底层的bean没有依赖，所以只有实例 + 初始)
- 初始化：依赖注入后到Bean完全可以使用之前
  - 依赖注入只会完成需要`@Autowire`、`setter`、`构造方法依赖`Bean的注入;也许Bean中还有其他属性需要初始化
  - 对于动态代理类，原始被代理的对象应该是一个基本功能完备的，例如Aware、其他前置处理都完成的Bean

### Context容器初始化流程

**容器初始化流程总结**
- BeanFactory初始化
- 装载Beandefinition，执行BeanFactoryPostProcessor，用以修改Bean定义
- 装载BeanPostProcessor，用以完成Bean初始化前后的操作
- 完成Bean的初始化、依赖注入、实例化
- 完成容器刷新，发送Event

_容器初始化整体源码流程_

[AbstractApplicationContext.java]
```java
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
		// Prepare this context for refreshing.
		prepareRefresh();

		// Tell the subclass to refresh the internal bean factory.
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

		// Prepare the bean factory for use in this context.
		prepareBeanFactory(beanFactory);

		try {
			// Allows post-processing of the bean factory in context subclasses.
			postProcessBeanFactory(beanFactory);

			// Invoke factory processors registered as beans in the context.
			invokeBeanFactoryPostProcessors(beanFactory);

			// Register bean processors that intercept bean creation.
			registerBeanPostProcessors(beanFactory);

			// Initialize message source for this context.
			initMessageSource();
	
			// Initialize event multicaster for this context.
			initApplicationEventMulticaster();

			// Initialize other special beans in specific context subclasses.
			onRefresh();

			// Check for listener beans and register them.
			registerListeners();

			// Instantiate all remaining (non-lazy-init) singletons.
			finishBeanFactoryInitialization(beanFactory);

			// Last step: publish corresponding event.
			finishRefresh();
		}
	}
}
```

### Bean的创建流程

_Bean创建整体源码流程_

[AbstractAutowireCapableBeanFactory.java]
```java
// 创建Bean的主流程
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {

	// 实例化Bean对象.
	BeanWrapper instanceWrapper = null;
	if (mbd.isSingleton()) {
		instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
	}
	if (instanceWrapper == null) {
		instanceWrapper = createBeanInstance(beanName, mbd, args);
	}
	Object bean = instanceWrapper.getWrappedInstance();

	// .... //

	// 初始化Bean对象.
	Object exposedObject = bean;
	try {
        // 注入属性
		populateBean(beanName, mbd, instanceWrapper);
        // 初始化Bean，理解为调用 init 方法、动态代理增强
		exposedObject = initializeBean(beanName, exposedObject, mbd);
	}

	// .... //
	return exposedObject;
}

// 初始化Bean
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
	// 注入Bean需要的Aware
    invokeAwareMethods(beanName, bean);

	Object wrappedBean = bean;
    // 执行BeanPostProcessor的before方法，返回被增强或处理的Bean对象（会执行AbstractAutoProxyCreator）
	if (mbd == null || !mbd.isSynthetic()) {
		wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
	}

	// 执行init方法（先执行afterPropertiesSet 再执行init）
	invokeInitMethods(beanName, wrappedBean, mbd);

    // 执行BeanPostProcessor的after方法（会执行AbstractAutoProxyCreator）
	wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);

	return wrappedBean;
}
```

### Spring 循环依赖

### Spring 常用扩展点

----

## Spring Boot