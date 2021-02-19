# Spring

## IOC/DI/AOP

### IOC: 控制反转
- Spring作为IOC容器，可以理解为一个很大的Map对象
- Map对象中维持对象名称和实例的映射

### DI: 依赖注入
- IOC是一种思想，将Bean和Bean之间的关系，通过容器进行维护
- DI是实现这种思想的一种手段

### AOP: 面向切面编程
**目的:** 提高代码复用，将日志、拦截增强等操作进行抽象

#### 实现方式:
**静态代理: 编译器实现增强，如Aspect**

**动态代理: 运行期实现增强，生成动态代理类**
- JDK动态代理:
    - 基于接口实现。原因: 基于JDK动态代理生成的类，默认继承Proxy类，Java是不支持多继承
    - 通过 `ProxyGenerator.generateProxyClass` 生成动态代理类字节码, 通过native的findClass方法装载Class到Metaspace区
- CGLIB动态代理:
    - 基于类实现。生成的动态代理类直接继承被代理类，完成增强环绕。所以，final方法是无法被代理的
    - 基于ASM改写字节码

-----

## Spring AOP

**实现方式:**
- 在

**参考:**
> 从动态代理实现到Spring AOP看这一篇就够了: https://mp.weixin.qq.com/s/EZ62B31Wxy47ISpiEKKVlg

----