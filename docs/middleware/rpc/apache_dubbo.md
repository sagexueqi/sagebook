# Apache Dubbo

## Dubbo线程模型

![Dubbo_线程模型](./images/dubbo_线程模型.png)

Dubbo底层默认使用Netty进行网络通信，Provider使用Netty的两次线程池，即`EventLoopGroup(Boss)`基于IO多路复用处理客户端的连接请求，完成TCP三次握手后将连接分发给`EventLoopGroup(Worker)`来处理。

> boss和worker线程我们统称为IO线程，前者为IO连接事件处理线程、后者为IO读写事件处理线程

### Dubbo线程分发策略

Dubbo线程模型是Dubbo可以实现高性能的基础，通过将IO线程池和逻辑线程池的隔离，提高整个中间件框架的并发能力。

如果服务是计算密集型或者逻辑较为简单，没有发起新的IO请求，我们完全可以将请求在`woker(IO读写处理线程)`中进行处理，这样也避免了线程池调度与上下文切换的开销，毕竟一切线程的切换都是有成本的。

我们绝大多数服务都是逻辑较为复杂，且需要发起新的网络通信，则请求必须派发到工作线程池中进行处理，否则IO线程被阻塞，导致`IO连接事件`无法快速分派，进而影响整个框架的IO请求效率。

Dubbo基于不同的线程分发策略，来决定请求在不同的线程池进行处理，以最大化的提升服务性能。

所有的`write`事件全部由`IO读写事件`处理线程完成

#### Dubbo内置的线程分发策略

**all:** 默认分发策略，消息所有处理都派发到dubbo线程池中处理，包括`connected`;`disconnected`;`received`;`caught`

- 实现类：`org.apache.dubbo.remoting.transport.dispatcher.all.AllChannelHandler`
- 适用于绝大多数场景，也是dubbo的默认线程分发策略
- 如果服务业务逻辑较复杂，耗时较长不建议使用。一旦服务接收大量请求，导致业务线程池被打满，此时抛出的拒绝异常会进入到`caught`方法进行处理；而`caught`方法也使用Dubbo线程池，导致逻辑无法处理，`Consumer`只能死等到超时才行

**direct:** 所有消息都不派发到线程池，全部在 IO 线程上直接执行

- 实现类：`org.apache.dubbo.remoting.transport.dispatcher.direct.DirectChannelHandler`
- 适合简单逻辑场景，尤其是业务逻辑中没有IO操作，只有一些内存操作的场景

_代码实现：_
```java
public class DirectChannelHandler extends WrappedChannelHandler {

    public DirectChannelHandler(ChannelHandler handler, URL url) {
        super(handler, url);
    }

    // 没有被重写的方法（connected、disconnected、caught），默认都是使用IO线程进行操作
    // 在all策略中，所有方法都会被重写，并获取DUBBO线程池进行操作

    @Override
    public void received(Channel channel, Object message) throws RemotingException {
        ExecutorService executor = getPreferredExecutorService(message);
        // 这里是针对Consumer的处理逻辑
        if (executor instanceof ThreadlessExecutor) {
            try {
                executor.execute(new ChannelEventRunnable(channel, handler, ChannelState.RECEIVED, message));
            } catch (Throwable t) {
                throw new ExecutionException(message, channel, getClass() + " error when process received event .", t);
            }
        } else {
            // 直接使用IO线程进行操作
            handler.received(channel, message);
        }
    }
}
```

**message:** 只有请求响应类消息派发到业务线程池，其他消息如连接事件、断开事件、心跳事件等，直接在I/O线程上执行。

- 实现类：`org.apache.dubbo.remoting.transport.dispatcher.message.MessageOnlyChannelHandler`
- 服务提供的业务逻辑较为复杂且耗时较长时建议选择；当大量请求进入到业务线程池后，即使打满线程池，`caught`是在IO线程进行处理的，避免`Consumer`一直等待进而引起系统雪崩

**execution:** 只把请求类消息派发到业务线程池处理，但是响应、连接事件、断开事件、心跳事件等消息直接在I/O线程上执行

- 实现类：`org.apache.dubbo.remoting.transport.dispatcher.execution.ExecutionChannelHandler`

**connection:** 在I/O线程上将连接事件、断开事件放入队列，有序地逐个执行，其他消息派发到业务线程池处理

- 实现类: `org.apache.dubbo.remoting.transport.dispatcher.connection.ConnectionOrderedChannelHandler`

**参考:**

> Dubbo线程模型: https://zhuanlan.zhihu.com/p/157354148
>
> Dubbo线程模型和调度策略: https://juejin.cn/post/6844903848159477774

----

## Dubbo为何高性能

----

## Dubbo服务上下线策略

----

## Dubbo与Spring的整合

----

## Dubbo与Spring Cloud体系

**参考**

> Dubbo——服务调用、服务暴露、服务引用过程: https://www.jianshu.com/p/15e77db72b75
>
> 优雅停机: https://dubbo.apache.org/zh/docs/v2.7/user/examples/graceful-shutdown/