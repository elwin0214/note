#Xmemcached-client 源码分析


##主要类

### XMemcachedClient
用于操作memcached。  

### CommandFactory
决定消息协议格式，默认使用TextCommandFactory。  

### Connector
管理tcp连接  
```
AbstractController  
-start()  
|  
NioController  
+selectorManager:SelectorManager  
-start0()  
|  
SocketChannelController  
|  
MemcachedConnector  
```

### Session 
tcp连接
```
AbstractSession  
+writeQueue:Queue<WriteMessage> #保存要发送到server 的command(get/set 等)  
-start() # 把连接注册到 selectorManager  
|  
AbstractNioSession:AbstractSession  
|  
-onWrite(SelectionKey key)  
-onRead(SelectionKey key)  
|  
NioTCPSession:AbstractNioSession  
|  
MemcachedTCPSession:NioTCPSession  
```

### Reactor

管理tcp连接读写事件
一个reactor对应多个session，并且一个session只属于一个reactor，reactor是一个线程。
```
Reactor
+register:Queue<RegisterEvent>  #(Queue<Reactor.RegisterEvent>) SystemUtils.createTransferQueue();

```
### Command

具体的请求

```java
Command
+latch:CountDownLatch
```

## 主要逻辑


### XMemcachedClient构造

*   XMemcachedClient::buildConnector()  
初始化成员connector(MemcachedConnector的实例)

*   XMemcachedClient::start0()  
调用connector.start()  
调用NioController::initialSelectorManager 初始化并启动selector

*   connect(final InetSocketAddressWrapper inetSocketAddressWrapper)  
根据connectionPoolSize 调用MemcachedConnector::connect()创建多个连接，注册到selectorManager进行管理


```java
->XMemcachedClient::XMemcachedClient()
    ->buildConnector()
        ->this.connector=newConnector()


    ->start0()
        ->startConnector()
            ->this.connector.start();
                ->NioController::initialSelectorManager() //初始化selector

    ->connect(final InetSocketAddressWrapper inetSocketAddressWrapper)
        ->MemcachedConnector::connect()
            ->MemcachedConnector::createSession(SocketChannel socketChannel, InetSocketAddressWrapper wrapper)
            ->MemcachedConnector::addSession()// 添加到connector ，根据一致性hash或者其他算法构造某种数据结构，便于根据key进行选择

```

### 线程相关

*  线程相关参数,见AbstractController::init()  
    * 设置read thread count （core pool size，下同），默认1，表示在异步线程池中处理，core线程数为1。  
    * 设置write thread count，默认0，表示在io线程（即Reactor）中处理。  
    * 设置msg thread count，默认0。  
    * 设置read write是否并发，默认true。  
    * 参数见Configuration 

*   线程池
    * 这里使用的线程池为PoolDispatcher，其中 maxPoolSize = 1.25 * corePoolSize

*   配置线程池，见AbstractController::start()  
    * setReadEventDispatcher  
    * setWriteEventDispatcher  
    * setDispatchMessageDispatcher  
    * 调用 NioController::start0() 启动 reactor  


### tcp连接读写

*   初次构造tcp连接时，把tcp连接注册到reactor    
    在client构造时，调用MemcachedConnector::createSession创建tcp连接，加入到reactor

```java

MemcachedConnector::createSession(SocketChannel socketChannel, InetSocketAddressWrapper wrapper)
    MemcachedConnector::buildSession(SocketChannel sc)
        MemcachedConnector::buildQueue()
            return new FlowControlLinkedTransferQueue(this.flowControl);
        构造一个 MemcachedTCPSession
    SelectorManager::registerSession(session, EventType.ENABLE_READ)
    session.start() //把连接注册到reactor
```

*   client 发送get command

```java
XMemcachedClient::get0()  //读取memcached中的数据
    XMemcachedClient::fetch0() 
        commandFactory.createGetCommand()  //创建一个command
        Session session = XMemcachedClient::sendCommand()  //发送command
            MemcachedConnector::send(command)// 使用connector 发送command
                MemcachedSession session = findSessionByKey()//这里会根据key查找对应的session
                session.write(msg)
        XMemcachedClient::latchWait(final Command cmd, final long timeout, final Session session)//等待server响应
            //如果超时抛出waiting for operation while connected to session异常
            cmd.getLatch().await(timeout, TimeUnit.MILLISECONDS)
```


*   command发送队列

```
->AbstractSession::write(Object packet)
    ->wrapMessage()
    ->AbstractNioSession::writeFromUserCode
        ->schduleWriteMessage(WriteMessage writeMessage)
          #写入到session 的 writeQueue中，
          如果 同reactor线程  return false；
          否则 发出一个写关注事件 
          SelectorManager::registerSession(session, EventType.ENABLE_WRITE)；return true
        ->onWrite(SelectionKey key)#如果 上一步false调用onWrite
```

```
->SelectorManager::registerSession(Session session, EventType event)
    ->找到session对应的reactor
    ->Reactor::registerSession(Session session, EventType event)
        -> 如果当前线程不是reactor线程，把{session,event}写入register，等reactor下一个回合注册事件处理
        ->Reactor::wakeup()
```

*   写完command对应的指令到TCP连接   
    写完command，添加到session 里面的 队列(commandAlreadySent)里面去

```java
AbstractNioSession::onWrite()
    ->MemcachedHandler::onMessageSent()
        ->this.commandAlreadySent.add(command);
```

*   command 有响应
    读到数据，找到发送完的cmd 列表取第一个，再处理

```java
AbstractNioSession::onEvent()
    ->readFromBuffer
        ->decodeAndDispatch
            ->MemcachedDecoder::decode
                ->MemcachedTCPSession::takeCurrentCommand
                    ->takeExecutingCommand
                        ->commandAlreadySent.take();
```


### 线程模块

线程有3种 

*   app线程：使用client的线程，比如：发送command。  
*   reactor线程：管理io读写事件线程，或者io读写。   
*   io读写线程：见线程池配置，如果没有单独配置线程池，则读写数据由reactor线程处理。  

## 流控
控制不需要响应的command数量

计算maxQueuedNoReplyOperations
以下2个取小值
int MAX_QUEUED_NOPS = 40000;
int DYNAMIC_MAX_QUEUED_NOPS = (int) (MAX_QUEUED_NOPS * (Runtime
            .getRuntime().maxMemory() / 1024.0 / 1024.0 / 1024.0));




## 注意
*   1.read操作（即使放在io读写线程中操作）没有加锁。
*   2.write操作连续写，对于服务器的响应的msg，直接call 之前写入的command，并不会写入一个command指令后，阻塞等待服务器输出。


