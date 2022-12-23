# Dubbo自定义异常扩展

**场景**

在服务的Provider内部的业务逻辑中，抛出了自定义的RuntimeException，但Consumer中无法catch；catch到的是`RuntimeException`

**解决方案**

- 在API方法签名上注明抛出的异常: `throws BussinesException`
- 扩展Dubbo默认的`org.apache.dubbo.rpc.filter.ExceptionFilter`，针对于自定义的Exception，不包装为RuntimeException抛出

**实现过程**

- 扩展Dubbo默认ExceptionFilter

```java
import org.apache.dubbo.common.constants.CommonConstants;
import org.apache.dubbo.common.extension.Activate;
import org.apache.dubbo.common.utils.ReflectUtils;
import org.apache.dubbo.common.utils.StringUtils;
import org.apache.dubbo.rpc.*;
import org.apache.dubbo.rpc.filter.ExceptionFilter;
import org.apache.dubbo.rpc.service.GenericService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.lang.reflect.Method;
import java.util.Arrays;
import java.util.List;

@Activate(group = CommonConstants.PROVIDER)
public class MuseDubboExceptionFilter extends ExceptionFilter {
    private Logger logger = LoggerFactory.getLogger(MuseDubboExceptionFilter.class);

    private static final List<String> IGNORE_LOG_CLAZZ = Arrays.asList("org.apache.dubbo.metadata.MetadataService");

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        return invoker.invoke(invocation);
    }

    @Override
    public void onResponse(Result appResponse, Invoker<?> invoker, Invocation invocation) {
        if (appResponse.hasException() && GenericService.class != invoker.getInterface()) {
            try {
                Throwable exception = appResponse.getException();

                // print WARN message in server's log
                if (!IGNORE_LOG_CLAZZ.contains(invoker.getInterface().getName()))
                    logger.warn("=> throw interface_name:{}, method_name:{} exception ",
                            invoker.getInterface().getName(), invocation.getMethodName(), exception);

                // directly throw if it's checked exception
                if (!(exception instanceof RuntimeException) && (exception instanceof Exception)) {
                    return;
                }
                // directly throw if the exception appears in the signature
                try {
                    Method     method           = invoker.getInterface().getMethod(invocation.getMethodName(), invocation.getParameterTypes());
                    Class<?>[] exceptionClasses = method.getExceptionTypes();
                    for (Class<?> exceptionClass : exceptionClasses) {
                        if (exception.getClass().equals(exceptionClass)) {
                            return;
                        }
                    }
                } catch (NoSuchMethodException e) {
                    return;
                }

                // for the exception not found in method's signature, print WARN message in server's log.
                // logger.warn("Got unchecked and undeclared exception which called by " + RpcContext.getServiceContext().getRemoteHost() + ". service: " + invoker.getInterface().getName() + ", method: " + invocation.getMethodName() + ", exception: " + exception.getClass().getName() + ": " + exception.getMessage(), exception);

                // directly throw if exception class and interface class are in the same jar file.
                String serviceFile   = ReflectUtils.getCodeBase(invoker.getInterface());
                String exceptionFile = ReflectUtils.getCodeBase(exception.getClass());
                if (serviceFile == null || exceptionFile == null || serviceFile.equals(exceptionFile)) {
                    return;
                }
                // directly throw if it's JDK exception
                String className = exception.getClass().getName();
                if (className.startsWith("java.") || className.startsWith("javax.")) {
                    return;
                }
                // directly throw if it's dubbo exception
                if (exception instanceof RpcException) {
                    return;
                }

                // 扩展: if exception extends RPCException,BusinessException,ValidException
                // 业务系统自定义的RuntimeException
                if (exception instanceof RPCException
                        || exception instanceof BusinessException
                        || exception instanceof ValidException) {
                    return;
                }

                // otherwise, wrap with RuntimeException and throw back to the client
                appResponse.setException(new RuntimeException(StringUtils.toString(exception)));
            } catch (Throwable e) {
                logger.warn("Fail to ExceptionFilter when called by " + RpcContext.getServiceContext().getRemoteHost() + ". service: " + invoker.getInterface().getName() + ", method: " + invocation.getMethodName() + ", exception: " + e.getClass().getName() + ": " + e.getMessage(), e);
            }
        }

    }
}
```

- 编辑`org.apache.dubbo.rpc.Filter`，将自定义filter添加到Dubbo SPI扩展实现中

```
exceptionFilter=com.framework.dubbo.filter.MuseDubboExceptionFilter
```

- 配置provider的拦截器，移除默认的excpetion

```properties
dubbo.provider.filter = -exception,exceptionFilter
```

**实现原理**

Dubbo异常过滤器，针对特定的Exception会直接抛出:

- 不是RuntimeException的子类
- 方法签名指定的Exception
- JDK Excpetion
- Dubbo RpcException

从上面的逻辑得知，我们业务系统自定义的RuntimeException不在直接抛出的范围内，而是包装为RuntimeException返回给客户端；导致客户端无法直接捕获进行处理

所以，我们继承Dubbo原生的ExceptionFilter，增加对我们业务系统的自定义RuntimeException的直接抛出处理逻辑

**最佳实践**

- 封装一个BizRPCExcpetion, 业务系统内部的异常如果需要在服务间通讯时抛出，继承该类
- 扩展ExceptionFilter，针对BizRPCExcpetion，做直接抛出处理