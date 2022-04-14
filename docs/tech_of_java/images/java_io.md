# Java IO

IO模型就是说用什么样的通道进行数据的发送和接收，首先要明确一点：IO是操作系统与其他网络进行数据交互，JDK底层并没有实现IO，而是对操作系统内核函数做的一个封装，IO代码进入底层其实都是native形式的。

## IO的基础概念

- 在unix系统中，所有操作皆为文件，不管socket,还是FIFO、管道、终端，对我们来说，一切都是文件，一切都是流
- 在信息交换的过程中，我们都是对这些流进行数据的收发操作，简称为I/O操作
- 假设启动服务端端口6380，执行 ls -l /proc/进程号/fd，观察描述符文件：

![javaio_proc_fd](./images/javaio_proc_fd.png)

- 启动一个客户端，连接到Server的6380端口，此时再次执行 ls -l /proc/进程号/fd，观察描述符文件：

![javaio_proc_fd_1](./images/javaio_proc_fd_1.png)

> 服务启动后，创建了socket描述符文件 4 -> socket:[2196628]
>
> 客户端启动连接到服务端后，创建了socket描述符文件 8 -> socket:[2196444]，这个文件表示连接到当前服务的socket连接
>
> 而这个文件的读写流可以抽象为**Channel**

### 同步、异步、阻塞、非阻塞

- **同步、异步：** 针对调用结果来说的。同步表示结果会在本次调用后等待返回；非同步是基于异步线程回调机制完成结果的通知
- **阻塞、非阻塞：** 针对服务端连接线程来说。阻塞表示线程接受一个连接后，直到最后返回，都不能再接受其他的连接；而非阻塞表示线程接受到连接后，交给后续线程处理，继续接受请求

### BIO NIO AIO

- **BIO:** 同步阻塞IO。同步并阻塞，服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器端就需要启动一个线程进行处理，如果这个连接不做任何事情会造成不必要的线程开销。

- **NIO:** 同步非阻塞IO。服务器实现模式为把多个连接(请求)放入集合中，只用一个线程可以处理多个请求(连接)，也就是多路复用。

- **AIO:** 异步非阻塞IO（异步一定是非阻塞的）

----

## IO多路复用

### Java实现IO多路复用(select/poll)模型

```java
public class Server {

    // 维持Socket连接的Channel
    private static final List<SocketChannel> SOCKET_CHANNEL_LIST = new ArrayList<>();
    // 数据缓冲区
    private static final ByteBuffer          BYTE_BUFFER         = ByteBuffer.allocate(1024);

    public static void main(String[] args) throws IOException {
        ServerSocketChannel serverSocket  = ServerSocketChannel.open();
        SocketAddress       socketAddress = new InetSocketAddress("127.0.0.1", 6379);
        serverSocket.bind(socketAddress);
        serverSocket.configureBlocking(false);  // 指定accept方法是否阻塞

        // 主线程轮询已连接的Socket，并在轮询中监听client的连接
        while (true) {
            for (int i = 0; i < SOCKET_CHANNEL_LIST.size(); i++) {
                    // 问题：如果Channel中均没有数据，会造成空轮询!!!
                    SocketChannel socketChannel = SOCKET_CHANNEL_LIST.get(i);

                    int read = socketChannel.read(BYTE_BUFFER);
                    if (read > 0) {
                        // 表示当前连接已经接收到数据,翻转ByteBuffer，准备读取数据
                        // 将limit指针指向当前位置，position指针设置为0
                                        BYTE_BUFFER.flip();
                        byte[] bs = new byte[read];
                                        BYTE_BUFFER.get(bs);
                                        BYTE_BUFFER.flip();
                                        
                        // 模拟执行业务流程
                        String content = new String(bs);
                                        System.out.println(String.format("socket:%s, read:{%s}", i, content));
                    }
            }

            // 非阻塞的接受新连接
            SocketChannel socketChannel = serverSocket.accept();
            if (socketChannel != null) {
                socketChannel.configureBlocking(false); // 指定read方法非阻塞
                SOCKET_CHANNEL_LIST.add(socketChannel);
                System.out.println("new connection, socket_channel_list_size:" + SOCKET_CHANNEL_LIST.size());
            }
        }
    }
}
```

- 上面的流程基于Java实现了简单的select模型
- 如果连接列表中的Channel均没有数据，会产生空轮询的问题

**对IO多路复用的一些误解**

- IO多路复用不是所有连接共用一个Socket：IO多路复用仍然是多个Socket，只不过是操作系统用一个线程一起监听他们的事件
- IO多路复用不是没有阻塞：其实IO多路复用的关键API调用(select，poll，epoll_wait）总是Block的
> 例如 Java NIO编程中，selector.select() 就是阻塞的
- IO多路复用没有减少IO的次数：实际上，IO本身（网络数据的收发）无论用不用IO多路复用和NIO，都没有变化。请求的数据该是多少还是多少；网络上该传输多少数据还是多少数据。IO多路复用仅仅是解决了一个调度的问题，减少线程的数量，降低系统资源的消耗

### IO多路复用的三种形式：select、poll、epoll

select、poll、epoll均为操作系统内核层面提供的能力。其中unix和windows提供select和poll的能力；epoll只有Linux系统提供

**select()**

- select的实现和上面Java代码的模式类似，知道有IO事件的发生，但是不知道是拿几个流(可能是1个 多个 或者全部)
- 需要无差别的 _轮询_ ，找到可以读取数据或者有写入数据的连接的连接，对他们进行操作
- O(n)级别的时间复杂度，连接越多，处理效率越低
- 有最大连接数限制，基于数组实现

**poll()**

- poll和select在本质上没有区别，将数据从网卡拷贝入内核态
- poll没有最大连接数限制，维持连接是基于 _链表_ 实现的

**epoll()**

- Linux内核所特有的模式
- epoll是基于事件驱动的event poll poll，epoll会把哪个流产生了什么样的IO事件通知给我们
- 因为是基于事件的，不同于select和poll，不会有空轮询的情况发生，事件复杂度为O(1)
- epoll最大的优点就在于它只管你“活跃”的连接

> 从字面概念来看，epoll的性能是优于select和poll的，但是在活跃连接数较小的情况下select和poll的性能可能会更好，毕竟epoll的通知会回调更多的函数

----

## Java NIO 

### Java NIO编码范式

**Server端:**

```java
public class NIOServer {
	// Selector: 选择器
	private static Selector selector;

	public static void main(String[] args) throws IOException {
		// url: https://developer.51cto.com/art/201112/307685.htm

		start();

		listener();
	}

	// 开启server
	private static void start() throws IOException {
		// 初始化服务器配置
		ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
		serverSocketChannel.bind(new InetSocketAddress("127.0.0.1", 6379));
		serverSocketChannel.configureBlocking(false);   // accept非阻塞

		// 初始化NIO多路复用选择器
		selector = Selector.open();
		// 将服务端的Channel绑定到selector中，并对ON_ACCEPT事件关心(接受客户端连接事件)
		// 对于Unix系统而言，所有的Socket也是一个文件，我们可以将Channel理解成对于Socket套接字文件的抽象
		// Server启动，也有对应的套接字文件等待连接
		serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
		System.out.println("start server success.");
	}

	// 监听事件并处理IO
	private static void listener() throws IOException {
		while (true) {
			// 调用操作系统poll or epoll 获取IO已经就绪的链接
			selector.select();
			// selector根据注册关心的事件，将就绪的channel封装在selectionKey中并返回
			Set<SelectionKey> selectionKeys = selector.selectedKeys();

			Iterator<SelectionKey> iterator = selectionKeys.iterator();
			while (iterator.hasNext()) {
				// 如果有就绪的channel，遍历处理
				SelectionKey selectionKey = iterator.next();

				// 要将channel移除本次iterator中
				// 如果没有remove掉 下一次select的时候 仍会返回上一次就绪的channel
				iterator.remove();

				// 如果是OP_ACCEPT-接入事件，表示有客户端连接到服务端
				if (selectionKey.isAcceptable()) {
					handlerAccept(selectionKey);
					continue;
				}
				// 如果是OP_READ-可读事件，表示有channel写入数据
				if (selectionKey.isReadable()) {
					handlerRead(selectionKey);
					continue;
				}
			}
		}
	}

	/**
	 * OP_ACCEPT事件channel处理，针对服务端开放端口的Socket文件Channel
	 *
	 * @param selectionKey
	 */
	private static void handlerAccept(SelectionKey selectionKey) throws IOException {
		// 获取服务端开放端口的Socket文件Channel
		ServerSocketChannel server = (ServerSocketChannel) selectionKey.channel();
		// 接受连接，并获取客户端连接的Socket文件Channel
		SocketChannel channel = server.accept();
		channel.configureBlocking(false);   // 设置read为非阻塞

		System.out.println("server accpet client connection.");

		// 将客户端连接Socket文件的Channel注册到selector中，并关心OP_READ事件
		// 将channel注册到selector时，可以写入附件信息，比如client_id
		Map<String, Object> attachment = new HashMap<>();
		attachment.put("client_id", UUID.randomUUID().toString());
		channel.register(selector, SelectionKey.OP_READ, attachment);
	}

	/**
	 * OP_READ事件channel处理，针对客户端连接Socket文件的Channel
	 *
	 * @param selectionKey
	 */
	private static void handlerRead(SelectionKey selectionKey) throws IOException {
		SocketChannel channel = (SocketChannel) selectionKey.channel();

		// 获取channel的附件信息
		Map<String, Object> attachment = (Map<String, Object>) selectionKey.attachment();

		ByteBuffer buffer = ByteBuffer.allocate(1024);
		if (channel.read(buffer) == -1) {
			// OP_READ阶段没有读取到数据，关闭连接
			channel.shutdownOutput();
			channel.shutdownInput();
			channel.close();
		} else {
			// 翻转buffer，准备读取
			buffer.flip();
			// 按照字符编码读取数据
			String receive = Charset.forName("utf-8").newDecoder().decode(buffer).toString();
			System.out.println("receive message. client:" + attachment.get("client_id") + ", receive:" + receive);
			buffer.clear();

			// 写出数据
			buffer = buffer.put("server received.".getBytes("utf-8"));
			buffer.flip();
			channel.write(buffer);
		}
	}
}
```
**Client端：**

```java
public class NIOClient {

	public static void main(String[] args) {
		try (SocketChannel socketChannel = SocketChannel.open()) {
			//连接服务端socket
			socketChannel.connect(new InetSocketAddress("127.0.0.1", 6379));

			// 从控制台获取输入
			Scanner    scanner = new Scanner(System.in);
			ByteBuffer buffer  = ByteBuffer.allocate(1024);
			while (true) {
				String line = scanner.nextLine();

				// 发送数据到服务端
				buffer.clear();
				buffer.put(line.getBytes("utf-8"));
				buffer.flip();
				socketChannel.write(buffer);

				// 读取服务器响应
				buffer.clear();
				int readLenth = socketChannel.read(buffer);
				if (readLenth < 0) {
					continue;
				}

				// 读取模式
				buffer.flip();
				byte[] bytes = new byte[readLenth];
				buffer.get(bytes);
				System.out.println(new String(bytes, "UTF-8"));
				buffer.clear();
			}

		} catch (IOException e) {
			e.printStackTrace();
		}
	}

}
```

**解决TCP通信中粘包拆包的方案**

- 常见方案是在报文头中写入整个报文体的长度，读取数据时候根据长度来组装真正的流数据
- 报文开头需要有魔数，来标示这是支持的报文和表示这是报文头

### NIO中的关键概念

#### Buffer(缓冲区)

在Java NIO中，所有数据都要通过Buffer进行读写。是与Channel进行交互的通道

![nio_buffer](./images/nio_buffer.png)

- position：Buffer当前指针位置
- limit：Buffer缓冲器数据的大小
- capacity：缓冲区容量，一旦设置不可修改

**HeapByteBuffer(堆内存Buffer)：** 

- 由JVM进行内存管理。当数据进行发送时，会拷贝到直接内存，再进行后续操作（例如sendfile等）
- 申请方式：``` ByteBuffer.alloc(1024); ```
- **_使用场景：_**
    - 数据只在Java进程内传输使用，不与本地进行IO（例如磁盘、网卡）
    - 数据小，对GC负担小

**DirectByteBuffer(堆外直接内存Buffer)：**

```
       Java        |      native
                   |
 DirectByteBuffer  |      malloc
 [    address   ] -+-> [   data    ]
```   
- 申请方式：``` ByteBuffer.allocDirect() ```
- 内存仍由JVM管理：
    - 由Full GC进行回收，通过System.gc()通知JVM在适当的时候进行回收
    - 创建DirectByteBuffer后，仍然会在堆内存中创建对象，记录着指向直接地址的address
    - **必须先释放堆内对象的引用，GC时才会去释放直接地址的空间**
    - 这里就很有可能由于对象引用没有被释放、多次YGC后进入老年代；导致对外内存无法回收，进而引发OOM
- **_使用场景：_**
    - 频繁的native IO，即java程序与本地磁盘、socket传输数据
    - DirectByteBuffer不会占用堆内存，也就是不会受到堆大小限制，只在DirectByteBuffer对象被回收后才会释放该缓冲区
    - 由于DirectByteBuffer的Cleaner是基于System.gc()的方式通知回收，所以JVM参数不要增加 ```-XX:+DisableExplicitGC``` - 禁止显示调用gc

#### Channel(通道)
- 类似于Java IO中流的概念，但流是单向的，而Channel是全双工(双向)工作的
- 通过Buffer读取和写出数据
- Channel可以理解为Unix系统中，Socket文件的抽象(/proc/进程号/fd)

#### Selector(选择器)
- 通过单线程来轮询符合响应事件的Channel的集合
- 可以理解为 select() 或 epoll() 的抽象
- Selector.select的过程一定是阻塞的，底层的调用实现因不同的操作系统而实现不同
- 基于Selector实现NIO多路复用

----

## Zero Copy 零拷贝技术

### 零拷贝是什么？

- 零拷贝技术并不是说数据没有拷贝和上下文切换的过程
- 目的：为了避免数据在 **用户态（User space）** 和 **内核态（Kernel space）**中来回拷贝的技术
- 减少双态间的拷贝，即减少了CPU的切换，进而减少CPU为数据在内存之间的拷贝而消耗运算能力


**参考**

> 深入理解BIO、NIO、AIO线程模型: https://blog.51cto.com/u_15281317/3007764#3_AIO_346
>
> Java NIO中Write事件和Connect事件: https://segmentfault.com/a/1190000017777939
>
> BIO,NIO与AIO的区别: https://www.cnblogs.com/barrywxx/p/8430790.html
>
> DirectByteBuffer堆外内存申请、回收: https://blog.csdn.net/liuxiao723846/article/details/89814005
>
> Java NIO - Buffer: https://yasinshaw.com/articles/54