# Spring Core

## Spring核心概念

**IOC:** 控制反转

- Spring作为IOC容器，可以理解为一个非常大的Map
- Map中维持着对象名称和实例之间的映射关系

**DI:** 依赖注入

- IOC是一种思想，将实例和名称维护在一个大的容器中，将Bean与Bean之间的关系，交由容器维护
- DI是实现这种思想的手段，依赖由容器注入到使用方中

**AOP:** 面向切面编程

- 目标：提高代码复用，将通用的日志、拦截增强等操作进行抽象
- 通过 _代理_ 方式实现AOP

**静态代理**

- 编译期实现增强，如`Aspect`

**动态代理**

- JDK动态代理
  - 基于接口实现。原因: 基于JDK动态代理生成的类，默认继承Proxy类，Java是不支持多继承
  - 通过 `ProxyGenerator.generateProxyClass` 生成动态代理类字节码, 通过native的findClass方法装载Class到Metaspace区
- CGLIB动态代理
  - 基于类实现。生成的动态代理类直接继承被代理类，完成增强环绕。所以，final方法是无法被代理的
  - 基于ASM改写字节码

----

## Spring AOP

可以说Spring所有的增强功能，都是基于Spring AOP实现的

**在Bean实例化完成并注入属性后，初始化Bean之前。执行`AbstractAutoProxyCreator`的`postProcessBeforeInstantiation`方法，返回通过AOP增强的动态代理类对象**

- 即执行`BeanPostProcessor`的`before`方法
- 如果有Advised增强作用于当前bean，例如@Transactional、自定义AOP环绕等，则通过JDK或CGLIB动态代理生成代理类；生成的代理类继承Advised接口,表示被增强；否则直接创建类或接口实现类对象本身

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

**如何使Spring AOP生效**
- 在方法调用时，必须通过被增强的Bean调用才可以，此时是使用的代理类对象
- 如果直接this.xxx()或者super.xxx()是无效的，因为该方法并没有被增强，还是走的内部调用。这是AOP不生效问题的常见原因。

----

## Spring Context

### 一些基本概念

- **实例化:** 创建Bean的原始对象，可以理解为只调用构造方法
- **依赖注入:** 根据BeanDefinition完成依赖Bean的`实例化 -> 注入 -> 初始化`(这是一个嵌套递归的过程，最底层的bean没有依赖，所以只有实例 + 初始化)
  - 构造方法注入(无法解决循环依赖)
  - Setter注入
  - @Autowaire注入 - 反射
- **初始化:** 依赖注入后到Bean完全可以使用之前
  - 依赖注入只会完成需要`@Autowire`、`setter`、`构造方法依赖`Bean的注入;也许Bean中还有其他属性需要初始化；初始化不是一个必须的过程，但是这个过程一定要有
  - 对于动态代理类，原始被代理的对象应该是一个基本功能完备的，例如Aware、其他前置处理都完成的Bean

### Context容器初始化流程

#### 容器初始化流程总结

- BeanFactory初始化
- 装载Beandefinition，执行BeanFactoryPostProcessor，用以修改Bean定义
- 装载BeanPostProcessor，用以完成Bean初始化前后的操作
- 完成Bean的实例化、依赖注入、初始化
- 完成容器刷新，发送Event事件

**容器初始化整体源码流程**

[AbstractApplicationContext.java]
```java
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
		// 容器刷新前的预处理操作：设置容器启动标示等操作.
		prepareRefresh();

		// 创建BeanFactory
		/*
		 * 1. createBeanFactory(): 创建BeanFctory
		 *
		 * 2. loadBeanDefinitions(): 装载BeanDefinition
		 */
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

		// BeanFactory使用前预处理
		/**
		 * 1. 设置BeanFactory的ClassLoader
		 *
		 * 2. 添加ApplicationContextAwareProcessor(BeanPostProcessor)：为实现Aware的Bean在初始化前注入所需的对象
		 *
		 * 3. 添加ApplicationListenerDetector：将实现ApplicationListener的Bean,在初始化后，通过BeanPostProcessor的After方法注册到Context中，为实现监听容器事件
		 */
		prepareBeanFactory(beanFactory);

		try {
			// BeanFactory后置处理，不常用.
			postProcessBeanFactory(beanFactory);

			// 执行BeanFactoryPostProcessors，以修改BeanDefinition（例如替换属性的value、移除某一些冲突的BeanDefinition）
			/**
			 * 1. 获取BeanDefinitionRegistryPostProcessor实现类的Bean并执行postProcessBeanDefinitionRegistry方法
			 * (一般来说我们可以直接实现该接口，这也是最常用的方式)
			 *
			 * 2. 获取直接实现BeanFactoryPostProcessor的Bean并执行postProcessBeanFactory方法
			 *
			 * 3. BeanFactoryPostProcessor的执行顺序，不受@Order注解控制；
			 * 优先排序并执行实现PriorityOrdered接口的Bean，然后再排序并执行实现Ordered接口的Bean
			 * 
			 * 4. BeanFactoryPostProcessor会通过BeanFactory的getBean方法提前实例化，但是不会注入属性和初始化，所以无法使用Bean
			 */
			invokeBeanFactoryPostProcessors(beanFactory);

			// 实例化并注册BeanPostProcessor实现类的Bean，用以扩展Bean初始化(init-method、afterPropertiesSet)前后的动作并返回被扩展的Bean对象
			/**
			 * 1. 要在实例化应用程序Bean之前，完成BeanPostProcessor实现类Bean的提前实例化，并注册到BeanFactory中
			 * 
			 * 2. 优先排序并执行实现PriorityOrdered接口的Bean，然后再排序并执行实现Ordered接口的Bean。同样不受@Order控制
			 *
			 * 3. 与BeanFactoryProcessor一样，Bean都是被提前实例化的，所以不能在实现类中注入应用程序Bean
			 */
			registerBeanPostProcessors(beanFactory);

			// 初始化消息处理，可理解为国际化支持
			initMessageSource();
	
			// 初始化容器事件广播组件，用来向各种ApplicationListener发送容器的Event
			initApplicationEventMulticaster();

			// 执行刷新方法，子类中实现刷新逻辑
			// 例如在Spring-boot-web中，在这个阶段创建createWebServer-web容器
			onRefresh();

			// 注册容器事件ApplicationListener监听器Bean
			/**
			 * 1. 将ApplicationListener实现类的BeanName注册到ApplicationEventMulticaster容器事件广播组件
			 * 
			 * 2. 使用beanName的方式，可以在监听器中使用应用程序Bean
			 */
			registerListeners();

			// 实例 + 初始化 非懒加载 的 应用程序Bean
			// beanFactory.preInstantiateSingletons() ---> AbstractBeanFactory.createBean() ---> AbstractAutowireCapableBeanFactory.doCreateBean() 完成
			finishBeanFactoryInitialization(beanFactory);

			// 完成容器初始化，并发布 ContextRefreshedEvent 事件.
			finishRefresh();
		}
	}
}
```

### Bean的初始化流程

![spring_bean初始化过程](./images/spring_bean初始化过程.png)

#### Bean初始化流程总结

- 实例化Bean对象：首先创建Bean对象实例，才可以进行后续的初始化和增强操作
- populateBean：根据BeanDefinition完成依赖属性注入
- 初始化Bean
  - 注入Aware对象
  - 执行BeanPostProcessor的`postProcessBeforeInitialization`方法: 目的是后续流程也许需要使用已经被proxy增强的功能，所以需要先执行
  - 执行afterPropertiesSet方法（此时已经可以使用被增强的Bean）
  - 执行init-method方法
  - 执行BeanPostProcessor的`postProcessAfterInitialization`方法

**Bean创建整体源码流程**

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
	// 为实现Aware接口的Bean，执行对应的注入属性或值的方法
	/**
	 * 1. 实现BeanNameAware接口的Bean，注入当前的BeanName
	 * 2. 实现BeanClassLoaderAware接口的Bean，注入当前的ClassLoader
	 * 3. 实现BeanFactoryAware接口的Bean，注入当前的BeanFactory
	 */
    invokeAwareMethods(beanName, bean);

	Object wrappedBean = bean;
	if (mbd == null || !mbd.isSynthetic()) {
		// 执行BeanPostProcessor的before方法
		/**
		 * 1. 所有BeanPostProcessor已经在registerBeanPostProcessors阶段，按照优先顺序添加到BeanFactory中
		 * 
		 * 2. 在这个阶段，调用AbstractAutoProxyCreator生成动态代理类对象（因为被代理对象已经被实例化了，可以createProxy了）
		 */
		wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
	}

	// 执行初始化逻辑方法（先执行afterPropertiesSet 再执行init）
	/*
	 * 1. 如果Bean实现了InitializingBean接口，首先执行afterPropertiesSet()方法
	 *
	 * 2. 执行Bean的init-method方法
	 */
	invokeInitMethods(beanName, wrappedBean, mbd);

    // 执行BeanPostProcessor的after方法（会执行AbstractAutoProxyCreator）
	wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);

	return wrappedBean;
}
```

##### Spring循环依赖

**什么是循环依赖?** Bean A依赖Bean B，Bean B中又依赖Bean A

**解决方案**

- 只有通过@Autowire和setter注入的方式的单例Bean可以解决
- 基于三级缓存的模式
- 构造器注入无法解决，不能通过构造方法创建早期对象
- 要尽量避免循环依赖，将代码结构拆散解耦才是主要解决方案

**参考:**
> 从动态代理实现到Spring AOP看这一篇就够了: https://mp.weixin.qq.com/s/EZ62B31Wxy47ISpiEKKVlg
>
> SpringBoot启动流程：https://www.processon.com/view/link/5f4c5d83e401fd14b2261d58?pw=src666