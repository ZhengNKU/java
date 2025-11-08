
https://dongzl.github.io/netty-handbook/#/_content/chapter02
##  NIO（Non-blocking I/O）框架
NIO 是 Java 提供的非阻塞 I/O 模型，解决了传统 BIO（Blocking I/O）的线程阻塞问题，适用于高并发场景。
**核心组件**：

- **Channel（通道）**  
    双向数据传输管道（如 `SocketChannel`、`FileChannel`），替代 BIO 的 `InputStream`/`OutputStream`。
    
- **Buffer（缓冲区）**  
    数据容器（如 `ByteBuffer`），通过 `flip()`、`clear()` 控制读写切换。
    
- **Selector（选择器）**  
    单线程监听多个 Channel 的 I/O 事件（如连接、读、写），实现多路复用。


### I/O模型
1. `I/O` 模型简单的理解：就是用什么样的通道进行数据的发送和接收，很大程度上决定了程序通信的性能。
2. `Java` 共支持 `3` 种网络编程模型 `I/O` 模式：`BIO`、`NIO`、`AIO`。
3. `Java BIO`：同步并阻塞（传统阻塞型），服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器端就需要启动一个线程进行处理，如果这个连接不做任何事情会造成不必要的线程开销。【简单示意图】
![[Pasted image 20250519201527.png]]
4. `Java NIO`：同步非阻塞，服务器实现模式为一个线程处理多个请求（连接），即客户端发送的连接请求都会注册到多路复用器上，多路复用器轮询到连接有 `I/O` 请求就进行处理。【简单示意图】
![[Pasted image 20250519201548.png]]
5. `Java AIO(NIO.2)`：异步非阻塞，`AIO` 引入异步通道的概念，采用了 `Proactor` 模式，简化了程序编写，有效的请求才启动线程，它的特点是先由操作系统完成后才通知服务端程序启动线程去处理，一般适用于连接数较多且连接时间较长的应用

#### I/O模型使用场景
1. `BIO` 方式适用于连接数目比较小且固定的架构，这种方式对服务器资源要求比较高，并发局限于应用中，`JDK1.4` 以前的唯一选择，但程序简单易理解。

<span style="background:rgba(136, 49, 204, 0.2)">流程：</span>
<span style="background:rgba(136, 49, 204, 0.2)">1. 服务器端启动一个 `ServerSocket`。</span>
<span style="background:rgba(136, 49, 204, 0.2)">2. 客户端启动 `Socket` 对服务器进行通信，默认情况下服务器端需要对每个客户建立一个线程与之通讯。</span>
<span style="background:rgba(136, 49, 204, 0.2)">3. 客户端发出请求后，先咨询服务器是否有线程响应，如果没有则会等待，或者被拒绝。</span>
<span style="background:rgba(136, 49, 204, 0.2)">4. 如果有响应，客户端线程会等待请求结束后，在继续执行。</span>

2. `NIO` 方式适用于连接数目多且连接比较短（轻操作）的架构，比如聊天服务器，弹幕系统，服务器间通讯等。编程比较复杂，`JDK1.4` 开始支持。</span></span>

<span style="background:rgba(136, 49, 204, 0.2)">特性：</span>
<span style="background:rgba(136, 49, 204, 0.2)">1. `NIO` 有三大核心部分: **`Channel`（通道）、`Buffer`（缓冲区）、`Selector`（选择器）** 。</span>
<span style="background:rgba(136, 49, 204, 0.2)">2. `NIO` 是**面向缓冲区，或者面向块编程**的。数据读取到一个它稍后处理的缓冲区，需要时可在缓冲区中前后移动，这就增加了处理过程中的灵活性，使用它可以提供非阻塞式的高伸缩性网络。</span>
<span style="background:rgba(136, 49, 204, 0.2)">3. `Java NIO` 的非阻塞模式，使一个线程从某通道发送请求或者读取数据，但是它仅能得到目前可用的数据，如果目前没有数据可用时，就什么都不会获取，而不是保持线程阻塞，所以直至数据变的可以读取之前，该线程可以继续做其他的事情。非阻塞写也是如此，一个线程请求写入一些数据到某通道，但不需要等待它完全写入，这个线程同时可以去做别的事情。通俗理解：`NIO` 是可以做到用一个线程来处理多个操作的。假设有 `10000` 个请求过来,根据实际情况，可以分配 `50` 或者 `100` 个线程来处理。不像之前的阻塞 `IO` 那样，非得分配 `10000` 个。</span>
<span style="background:rgba(136, 49, 204, 0.2)">4. `HTTP 2.0` 使用了多路复用的技术，做到同一个连接并发处理多个请求，而且并发请求的数量比 `HTTP 1.1` 大了好几个数量级。</span>

3. `AIO` 方式使用于连接数目多且连接比较长（重操作）的架构，比如相册服务器，充分调用 `OS` 参与并发操作，编程比较复杂，`JDK7` 开始支持。

### NIO和BIO的区别
1. `BIO` 以流的方式处理数据，而 `NIO` 以块的方式处理数据，块 `I/O` 的效率比流 `I/O` 高很多。
2. `BIO` 是阻塞的，`NIO` 则是非阻塞的。
3. `BIO` 基于字节流和字符流进行操作，而 `NIO` 基于 `Channel`（通道）和 `Buffer`（缓冲区）进行操作，数据总是从通道读取到缓冲区中，或者从缓冲区写入到通道中。`Selector`（选择器）用于监听多个通道的事件（比如：连接请求，数据到达等），因此使用单个线程就可以监听多个客户端通道。



### 三大核心
#### 缓冲区（buffer）
缓冲区（`Buffer`）：缓冲区本质上是一个**可以读写数据的内存块**，可以理解成是一个**容器对象（含数组）**，该对象提供了一组方法，可以更轻松地使用内存块，，缓冲区对象内置了一些机制，能够跟踪和记录缓冲区的状态变化情况。`Channel` 提供从文件、网络读取数据的渠道，但是读取或写入的数据都必须经由 `Buffer`
![[Pasted image 20250520100841.png]]

##### ByteBuffer常用方法
![[Pasted image 20250520100928.png]]

#### 通道（Channel）
1. `NIO` 的通道类似于流，但有些区别如下：
    - 通道可以同时进行读写，而流只能读或者只能写
    - 通道可以实现异步读写数据
    - 通道可以从缓冲读数据，也可以写数据到缓冲:
2. `BIO` 中的 `Stream` 是单向的，例如 `FileInputStream` 对象只能进行读取数据的操作，而 `NIO` 中的通道（`Channel`）是双向的，可以读操作，也可以写操作。
3. 常用的 `Channel` 类有: **`FileChannel`、`DatagramChannel`、`ServerSocketChannel` 和 `SocketChannel`** 。【`ServerSocketChanne` 类似 `ServerSocket`、`SocketChannel` 类似 `Socket`】
4. `FileChannel` 用于文件的数据读写，`DatagramChannel` 用于 `UDP` 的数据读写，`ServerSocketChannel` 和 `SocketChannel` 用于 `TCP` 的数据读写。

##### FileChannel常用方法
- `public int read(ByteBuffer dst)`，从通道读取数据并放到缓冲区中
- `public int write(ByteBuffer src)`，把缓冲区的数据写到通道中
- `public long transferFrom(ReadableByteChannel src, long position, long count)`，从目标通道中复制数据到当前通道
- `public long transferTo(long position, long count, WritableByteChannel target)`，把数据从当前通道复制给目标通道

#### 选择器（Selector）
1. `Netty` 的 `IO` 线程 `NioEventLoop` 聚合了 `Selector`（选择器，也叫多路复用器），可以同时并发处理成百上千个客户端连接。
2. 当线程从某客户端 `Socket` 通道进行读写数据时，若没有数据可用时，该线程可以进行其他任务。
3. 线程通常将非阻塞 `IO` 的空闲时间用于在其他通道上执行 `IO` 操作，所以单独的线程可以管理多个输入和输出通道。
4. 由于读写操作都是非阻塞的，这就可以充分提升 `IO` 线程的运行效率，避免由于频繁 `I/O` 阻塞导致的线程挂起。
5. 一个 `I/O` 线程可以并发处理 `N` 个客户端连接和读写操作，这从根本上解决了传统同步阻塞 `I/O` 一连接一线程模型，架构的性能、弹性伸缩能力和可靠性都得到了极大的提升。

6. `Selector` 相关方法说明
    - `selector.select();` //阻塞
    - `selector.select(1000);` //阻塞 1000 毫秒，在 1000 毫秒后返回
    - `selector.wakeup();` //唤醒 selector
    - `selector.selectNow();` //不阻塞，立马返还




## Netty 框架
基于 NIO 的**高性能网络应用框架**，封装了 NIO 的复杂性，提供易用的 API 和丰富功能。

### 原生 NIO 存在的问题

1. `NIO` 的类库和 `API` 繁杂，使用麻烦：需要熟练掌握 `Selector`、`ServerSocketChannel`、`SocketChannel`、`ByteBuffer`等。
2. 需要具备其他的额外技能：要熟悉 `Java` 多线程编程，因为 `NIO` 编程涉及到 `Reactor` 模式，你必须对多线程和网络编程非常熟悉，才能编写出高质量的 `NIO` 程序。
3. 开发工作量和难度都非常大：例如客户端面临断连重连、网络闪断、半包读写、失败缓存、网络拥塞和异常流的处理等等。4. `JDK NIO` 的 `Bug`：例如臭名昭著的 `Epoll Bug`，它会导致 `Selector` 空轮询，最终导致 `CPU100%`。直到 `JDK1.7` 版本该问题仍旧存在，没有被根本解决。


### Reactor 模式
主从 Reactor 多线程
![[Pasted image 20250521110115.png]]

1. `Reactor` 主线程 `MainReactor` 对象通过 `select` 监听连接事件，收到事件后，通过 `Acceptor` 处理连接事件
2. 当 `Acceptor` 处理连接事件后，`MainReactor` 将连接分配给 `SubReactor`
3. `subreactor` 将连接加入到连接队列进行监听，并创建 `handler` 进行各种事件处理
4. 当有新事件发生时，`subreactor` 就会调用对应的 `handler` 处理
5. `handler` 通过 `read` 读取数据，分发给后面的 `worker` 线程处理
6. `worker` 线程池分配独立的 `worker` 线程进行业务处理，并返回结果
7. `handler` 收到响应的结果后，再通过 `send` 将结果返回给 `client`
8. `Reactor` 主线程可以对应多个 `Reactor` 子线程，即 `MainRecator` 可以关联多个 `SubReactor`
优点：父线程与子线程的数据交互简单职责明确，父线程只需要接收新连接，子线程完成后续的业务处理。

### Netty模型
![[Pasted image 20250521113049.png]]

1. `BossGroup` 线程维护 `Selector`，只关注 `Accecpt`
2. 当接收到 `Accept` 事件，获取到对应的 `SocketChannel`，封装成 `NIOScoketChannel` 并注册到 `Worker` 线程（事件循环），并进行维护
3. 当 `Worker` 线程监听到 `Selector` 中通道发生自己感兴趣的事件后，就进行处理（就由 `handler`），注意 `handler` 已经加入到通道

### Netty核心模块组件
#### Bootstrap、ServerBootstrap
`Bootstrap` 意思是引导，一个 `Netty` 应用通常由一个 `Bootstrap` 开始，主要作用是配置整个 `Netty` 程序，串联各个组件，`Netty` 中 `Bootstrap` 类是客户端程序的启动引导类，`ServerBootstrap` 是服务端启动引导类。

常用方法
- `public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup)`，该方法用于服务器端，用来设置两个 `EventLoop`
- `public B group(EventLoopGroup group)`，该方法用于客户端，用来设置一个 `EventLoop`
- `public B channel(Class<? extends C> channelClass)`，该方法用来设置一个服务器端的通道实现
- `public <T> B option(ChannelOption<T> option, T value)`，用来给 `ServerChannel` 添加配置
- `public <T> ServerBootstrap childOption(ChannelOption<T> childOption, T value)`，用来给接收到的通道添加配置
- `public ServerBootstrap childHandler(ChannelHandler childHandler)`，该方法用来设置业务处理类（自定义的`handler`）
- `public ChannelFuture bind(int inetPort)`，该方法用于服务器端，用来设置占用的端口号
- `public ChannelFuture connect(String inetHost, int inetPort)`，该方法用于客户端，用来连接服务器端

#### Channel
不同协议、不同的阻塞类型的连接都有不同的 `Channel` 类型与之对应，常用的 `Channel` 类型：
    - `NioSocketChannel`，异步的客户端 `TCP` `Socket` 连接。
    - `NioServerSocketChannel`，异步的服务器端 `TCP` `Socket` 连接。
    - `NioDatagramChannel`，异步的 `UDP` 连接。
    - `NioSctpChannel`，异步的客户端 `Sctp` 连接。
    - `NioSctpServerChannel`，异步的 `Sctp` 服务器端连接，这些通道涵盖了 `UDP` 和 `TCP` 网络 `IO` 以及文件 `IO`。

#### Selector
1. `Netty` 基于 `Selector` 对象实现 `I/O` 多路复用，通过 `Selector` 一个线程可以监听多个连接的 `Channel` 事件。
2. 当向一个 `Selector` 中注册 `Channel` 后，`Selector` 内部的机制就可以自动不断地查询（`Select`）这些注册的 `Channel` 是否有已就绪的 `I/O` 事件（例如可读，可写，网络连接完成等），这样程序就可以很简单地使用一个线程高效地管理多个 `Channel`



