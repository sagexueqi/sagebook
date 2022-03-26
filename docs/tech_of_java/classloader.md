# ClassLoader(类加载器)

## JVM的类加载机制

### JVM的类加载器

**BootstrapClassLoader(引导类加载器)**: 加载 `jre/lib` 路径下的 class，以及所有`java.*`开头的class。BootstrapClassLoader是无法被Java程序直接引用的。

**ExtClassLoader(扩展类加载器)**: 加载 `jre/ext` 路径下的 class

**AppClassLoader(应用程序加载器)**: 加载工程下的class信息

### 双亲委派机制

#### 工作流程

**向上委托加载过程:**

- 当`AppClassLoader`加载一个Class时，它首先不会加载这个类，而是把这个类委托给父加载器`ExtClassLoader`完成
- 当`ExtClassLoader`加载一个Class时，它也不会加载这个类，而是把这个类委托给父级加载器`BootstrapClassLoader`

**向下加载过程:**
- `BootstrapClassLoader`如果加载失败，会向下委托`ExtClassLoader`尝试加载
- `ExtClassLoader`如果也加载失败，会继续向下委托`AppClassLoader`进行加载；如果`AppClassLoader`也加载失败，会抛出`ClassNotFoundException`（常见异常，会引出类加载机制问题）

> 通过URLClassLoader的findClass方法的描述，一个类先从本地进行加载，然后再打开JAR包进行加载

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

> **loadClass()和findClass()的区别**
>
> - 只重写findClass()：不打破双亲委派机制，已加载的class会被缓存
> - 重写loadClass()：自定义类加载的整个流程，可以打破双亲委派机制。有些场景，我们可能不需要缓存已经被加载的class，此时就需要重写loadClass()方法，去掉缓存的逻辑

#### 双亲委派的特点

- 类加载之间没有实际的继承关系，而是通过JDK源码强制指定的加载逻辑
- 安全: 例如`java.lang.String`类，如果没有双亲委托机制，直接从本地加载一个恶意的`java.lang.String`类，会造成安全问题。所以必须先委托到`BootstrapClassLoader`进行加载
- 避免重复加载: 如果某一个类已经被父级加载器加载，子加载器就没必要再次加载
- 不论是JVM类加载机制，还是Tomcat打破双亲委托的类加载机制，处于安全目的，都要先委托到Bootstrap ClassLoader进行类加载

> 对于JVM来说，只有两种类加载器。一种是`Bootstrap CLassloader - 启动类加载器` 和 所有其他类加载器。`Bootstrap`加载器作为虚拟机的一部分，基于C++实现；
>
> 而其他加载器均独立于虚拟机之外，并且都继承`java.lang.ClassLoader`类，这些加载器需要由Bootstrap加载器加载到内存后才可以使用，去加载其他类
>
> 对于加载和初始化的区别
>
>> 通过ClassLoader.loadClass方法，只做一件事，将类元信息加载到JVM的元空间中并生成Class对象，这是加载；通过Class.forName方法，默认在加载完成之后，会初始化Class，执行初始化块

----

## JVM类加载机制的特点

**全盘负责**: 
- 当一个类加载器负责加载某一个Class时，它的所有依赖都由当前加载器加载，除非显示指定

**父类委托** :
- 先让父级加载器尝试加载，只有父类加载器都无法加载该类的时候，才尝试从自己的路径中加载该类
- 针对一般Java项目，通过URLClassLoader的findClass方法的描述，一个类先从本地进行加载，然后再打开JAR包进行加载

**缓存机制**: 
- 当前加载器加载过的Class都会被缓存，当程序中需要某个Class时，类加载器优先从缓存中寻找该class，只有缓存不存在时才会加载二进制数据，并将其转换为Class对象，存入缓冲区。
- 这也就解释了，修改Class需要重启JVM；热部署或者其他场景，重复创建自定义ClassLoader时，会造成元空间泄露的原因

----

## Tomcat的类加载机制

### Tomcat类加载的特点

- 每一个应用都有独立的`WebAppClassLoader`，实现应用的ClassLoader隔离；通过`CommonClassloader`加载`TOMCAT_HOME/lib`下的class，以实现类共享
- 默认打破双亲委派机制，目的在于快速的加载应用中的类
- 通过`WebAppClassLoader`实现应用间的classloader隔离

### Tomcat类加载流程

**非双亲委派模型**

- 当`WebAppClassLoader`加载一个类时，首先向上委托给`ExtClassLoader`，再向上委托`BootstrapClassLoader`进行加载；此过程没有打破双亲委派机制流程，目的在于加载`JAVA_HOME/jre/lib`和`JAVA_HOME/jre/lib/ext`下的class，避免核心类被篡改
- `WebAppClassLoader`加载`WEB-INF/classes`中的class
- `WebAppClassLoader`加载`WEB-INF/lib`中jar包的class：_通过这一步，在日常开发中，如果以来的第三方工具包中有类需要替换，那我们可以在工程中定义一个同包同名的类，实现自己的逻辑_
- 向上委托`CommonClassLoader`加载，`CommonClassLoader`再基于双亲委派机制向上委托`SystemClassLoader`加载`TOMECAT_HOME/lib`等路径下的class：

**双亲委派模型**
- 当`WebAppClassLoader`加载一个类时，首先向上委托给`ExtClassLoader`，再向上委托`BootstrapClassLoader`进行加载；此过程没有打破双亲委派机制流程，目的在于加载`JAVA_HOME/jre/lib`和`JAVA_HOME/jre/lib/ext`下的class，避免核心类被篡改
- `WebAppClassLoader`向上委托`CommonClassLoader`加载，`CommonClassLoader`再基于双亲委派机制向上委托`SystemClassLoader`加载`TOMECAT_HOME/lib`等路径下的class：
- `WebAppClassLoader`加载`WEB-INF/classes`中的class
- `WebAppClassLoader`加载`WEB-INF/lib`中jar包的class

> Tomcat打破打破双亲委派模型是做用在加载应用内class的阶段，优先加载WEB-INF下的class，加速加载速度；在加载java核心class的阶段，并没有打破，这也是出于安全的考虑

**Tomcat类加载器关系**

```
            BootstrapClassloader
                     |
        SystemClassloader(ExtClassloader)
                     |
        CommonClassloader(TOMCAT_HOME/lib)
                /               \
    WebAppClassloader        WebAppClassloader
```

#### Tomcat类加载源码概要分析

```java
// Bootstrap类的init方法，初始化Tomcat体系下的相关ClassLoader
public final class Bootstrap {
    public void init() throws Exception {
        // todo: CL-01 初始化ClassLoader入口
        initClassLoaders();
    }
    // todo: CL-02 初始化Tomcat的公共类加载器
    private void initClassLoaders() {
        try {
            // todo：CL-02 创建CommonClassLoader
            commonLoader = createClassLoader("common", null);
            if (commonLoader == null) {
                // no config file, default to this loader - we might be in a 'single' env.
                commonLoader = this.getClass().getClassLoader();
            }
            // todo: CL-03 创建CatalinaClassLoader，默认也是用CommonClassLoader
            catalinaLoader = createClassLoader("server", commonLoader);
            // todo: CL-04 创建SharedClassLoader，默认也是用CommonClassLoader
            sharedLoader = createClassLoader("shared", commonLoader);
        } catch (Throwable t) {
            handleThrowable(t);
            log.error("Class loader creation threw exception", t);
            System.exit(1);
        }
    }
}

// StandardContext负责启动各个应用
public class StandardContext extends ContainerBase implements Context, NotificationEmitter {

    // todo: CL-06 各应用启动
    protected synchronized void startInternal() throws LifecycleException {
        // .... 省略其他逻辑 .... //

        // todo: CL-07 初始化各web应用的WebAppClassLoader
        if (getLoader() == null) {
            WebappLoader webappLoader = new WebappLoader();
            webappLoader.setDelegate(getDelegate());
            setLoader(webappLoader);
        }

        // .... 省略其他逻辑 .... //
    }

    public void setLoader(Loader loader) {
        // .... 省略其他逻辑 .... //

        // todo: CL-08 启动WebAppClassLoader
        ((Lifecycle) loader).start();

        // .... 省略其他逻辑 .... //
    }
}

// WebappLoader负责创建和启动WebappClassLoaderBase类加载器
public class WebappLoader extends LifecycleMBeanBase implements Loader, PropertyChangeListener {
    // todo: CL-09 启动WebAppClassLoader
    protected void startInternal() throws LifecycleException {
        // .... 省略其他逻辑 .... //

        // todo: CL-10 创建WebappClassLoaderBase
        classLoader = createClassLoader();
        classLoader.setResources(context.getResources());
        classLoader.setDelegate(this.delegate);

        // .... 省略其他逻辑 .... //
    }

    public void backgroundProcess() {
        // 热部署实现流程
    }
}

// 应用的类加载器实现
public abstract class WebappClassLoaderBase extends URLClassLoader implements Lifecycle, InstrumentableClassLoader, WebappProperties, PermissionCheck {

    // 启动逻辑，在这里决定了class资源的装载顺序是先 /WEB-INF/classes 再 /WEB-INF/lib
    @Override
    public void start() throws LifecycleException {

        state = LifecycleState.STARTING_PREP;

        // todo: CL-11 WebappClassLoaderBase加载/WEB-INF/classes下的资源
        WebResource[] classesResources = resources.getResources("/WEB-INF/classes");
        for (WebResource classes : classesResources) {
            if (classes.isDirectory() && classes.canRead()) {
                localRepositories.add(classes.getURL());
            }
        }

        // todo: CL-12 WebappClassLoaderBase加载/WEB-INF/lib下载jar的资源
        WebResource[] jars = resources.listResources("/WEB-INF/lib");
        for (WebResource jar : jars) {
            if (jar.getName().endsWith(".jar") && jar.isFile() && jar.canRead()) {
                localRepositories.add(jar.getURL());
                jarModificationTimes.put(
                        jar.getName(), Long.valueOf(jar.getLastModified()));
            }
        }

        state = LifecycleState.STARTED;
    }

    // todo: CL-13 重写loadClass 打破双亲委派加载类。所谓打破双亲委派，也是在加载应用和tomcat容器的class时。加载java core的class，还是要遵循双亲委派机制的流程以保证安全
    public Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        // (0) Check our previously loaded local class cache
        // todo: CL-14(0) 检查当前WebAppClassLoader是否已经加载过当前的class
        clazz = findLoadedClass0(name);

        // (0.1) Check our previously loaded class cache
        // todo: CL-14(0.1) 检查JVM虚拟机是否已经加载过当前class
        clazz = JreCompat.isGraalAvailable() ? null : findLoadedClass(name);

        // todo: CL-14(0.2)使用ExtClassLoader加载class，ExtClassLoader会向上委托到BootstrapClassLoader加载（这样可以避免/jre/lib和/jre/lib/ext的核心类被替换）
        ClassLoader javaseLoader = getJavaseClassLoader();
        clazz = javaseLoader.loadClass(name);

        // --- todo: CL-14 javase核心类的加载流程，还是符合双亲委托机制流程的 --- //

        // todo: CL-14(1.0) 使用双亲委托机制流程加载class
        if (delegateLoad) {
            // todo: CL-14(1.1) WebAppClassLoader先委托给parent加载器（CommonClassLoader）加载，按照 Common -> System 的顺序委托加载
            clazz = Class.forName(name, false, parent);
        }

        // todo: CL-14(2) 无论是不是双亲委托，上一步没有找到class，都要通过自己加载
        clazz = findClass(name);

        if (!delegateLoad) {
            // todo: CL-14(3) 不使用双亲委派流程时，WebAppClassLoader没有加载到class后，也要委托给parent加载器（CommonClassLoader）加载，按照 Common -> System 的顺序委托
            clazz = Class.forName(name, false, parent);
        }
    }
}
```

**参考**

> java类的加载机制: https://www.cnblogs.com/ityouknow/p/5603287.html
>
> Java ClassLoader实现热加载: https://my.oschina.net/ososchina/blog/1599977
>
> java利用classloader实现热部署: https://blog.csdn.net/chaofanwei2/article/details/51298818
>
> Apache Tomcat 9 - Class Loader How-To: https://tomcat.apache.org/tomcat-9.0-doc/class-loader-howto.html