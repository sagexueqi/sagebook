# ClassLoader

## JVM的类加载机制

### JVM的类加载器
**BootstrapClassLoader(引导类加载器)**: 加载 `jre/lib` 路径下的 class

**ExtClassLoader(扩展类加载器)**: 加载 `jre/ext` 路径下的 class

**AppClassLoader(应用程序加载器)**: 加载工程下的class信息

----

### JVM类加载机制的特点
**全盘负责**: 
- 当一个类加载器负责加载某一个Class时，它的所有依赖都由当前加载器加载，除非显示指定

**父类委托** :
- 先让父级加载器尝试加载，只有父类加载器都无法加载该类的时候，才尝试从自己的路径中加载该类
- 针对一般Java项目，通过URLClassLoader的findClass方法的描述，一个类先从本地进行加载，然后再打开JAR包进行加载

**缓存机制**: 
- 当前加载器加载过的Class都会被缓存，当程序中需要某个Class时，类加载器优先从缓存中寻找该class，只有缓存不存在时才会加载二进制数据，并将其转换为Class对象，存入缓冲区。
- 这也就解释了，修改Class需要重启JVM；热部署或者其他场景，重复创建自定义ClassLoader时，会造成元空间泄露的原因

----

### 双亲委派机制
**工作流程**
- 向上委托加载过程: 
  - 当`AppClassLoader`加载一个Class时，它首先不会加载这个类，而是把这个类委托给父加载器`ExtClassLoader`完成；
  - 当`ExtClassLoader`加载一个Class时，它也不会加载这个类，而是把这个类委托给父级加载器`BootstrapClassLoader`;

- 向下加载过程:
  - `BootstrapClassLoader`如果加载失败，会向下委托`ExtClassLoader`尝试加载
  - `ExtClassLoader`如果也加载失败，会继续向下委托`AppClassLoader`进行加载；如果`AppClassLoader`也加载失败，会抛出`ClassNotFoundException`（常见异常，会引出类加载机制问题）

> 针对一般Java项目，通过URLClassLoader的findClass方法的描述，一个类先从本地进行加载，然后再打开JAR包进行加载

**双亲委托的特性**
- 类加载之间没有实际的继承关系，而是通过JDK源码强制指定的加载逻辑

```java
public abstract class ClassLoader {
    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // 首先在本地缓存查询是否已经加载过该Class
            Class<?> c = findLoadedClass(name);
            // 如果没有加载过，使用双亲委派模型加载
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    // parent全局属性维持着当前classloader的上一级
                    // 如果不为空，表示有上级且不是bootstrap classloader
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        // 上一级classloader为空，调用native方法，直接由bootstrap加载器加载
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                // 为自定义classloader实现加载类的逻辑，因为即使是自定义classloader也要符合双亲委派模型
                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
}
```

- 安全考虑: 例如`java.lang.String`类，如果没有双亲委托机制，直接从本地加载一个恶意的`java.lang.String`类，会造成安全问题。所以必须先委托到`BootstrapClassLoader`进行加载

- 避免重复加载: 如果某一个类已经被父级加载器加载，子加载器就没必要再次加载

- 不论是JVM类加载机制，还是Tomcat打破双亲委托的类加载机制，处于安全目的，都要先委托到Bootstrap ClassLoader进行类加载

----

> 对于JVM来说，只有两种类加载器。一种是`Bootstrap CLassloader - 启动类加载器` 和 所有其他类加载器。`Bootstrap`加载器作为虚拟机的一部分，基于C++实现；
>
> 而其他加载器均独立于虚拟机之外，并且都继承`java.lang.ClassLoader`类，这些加载器需要由Bootstrap加载器加载到内存后才可以使用，去加载其他类
>
> 对于加载和初始化的区别
> 通过ClassLoader.loadClass方法，只做一件事，将类元信息加载到JVM中并生成Class对象，这是加载；通过Class.forName方法，默认在加载完成之后，会初始化Class，执行初始化块

----

## Tomcat的类加载机制

### Tomcat的类加载流程

**特点：** 
- 每一个应用都有独立的`WebAppClassLoader`，实现应用的ClassLoader隔离
- 默认打破双亲委派机制，目的在于快速的加载应用中的类

**加载流程**
- 当`WebAppClassLoader`加载一个类时，首先向上委托给`BootstrapClassLoader`进行加载：目的是加载java核心类
- 如果`BootstrapClassLoader`加载不到，`WebAppClassLoader`首先加载`WEB-INF/classes`中的类
- `WebAppClassLoader`其次加载`WEB-INF/lib`中jar包的类：_通过这一步，在日常开发中，如果以来的第三方工具包中有类需要替换，那我们可以在工程中定义一个同包同名的类，实现自己的逻辑_
- 通过`ExtClassLoader`加载`${JAVA_HOME}/jre/ext`路径下的类
- 通过`CommonClassLoader`加载`${TOMCAT_HOME}/lib`下的类: _通过这一步，tomcat实现了公共类的不隔离和应用间类的隔离_

- Tomcat默认使用上述打破双亲委派机制加载类，可以通过配置恢复为双亲委派机制
```
            BootstrapClassloader
                     |
        SystemClassloader(ExtClassloader)
                     |
        CommonClassloader(TOMCAT_HOME/lib)
                /               \
    WebAppClassloader        WebAppClassloader
```

----

### Tomcat热部署的实现流程
- Tomcat运行DEMOND线程，监听war包文件的变化
- 销毁当前的WebAppClassLoader，重新解压war包，并创建新的WebAppClassLoader
- 这样做的好处在于，不重复使用WebAppClassLoader。也避免由于类加载器没有卸载，导致元空间内存溢出的问题

----

参考:
> java类的加载机制: https://www.cnblogs.com/ityouknow/p/5603287.html