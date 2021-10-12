# Service Mesh数据面sidecar envoy源码分析-下游tcp连接处理
本文章主要介绍下游tcp执行connect后，envoy accect连接后，执行accect socket创建、accect fd注册到libevent的详细操作。对accect socket的读写事件响应暂不做介绍。
## listen fd响应读事件
前面文章讲到，当listen fd有读事件发生时，读事件会被其他（主动的）socket的连接所触发，即client的connect操作，程序会调用TcpListenerImpl::onSocketEvent()，然后调用TcpListenerImpl成员变量cb_的onAccept方法,cb_指向ConnectionHandlerImpl::ActiveTcpListener实例，所以最终调用ConnectionHandlerImpl::ActiveTcpListener::onAccept(Network::ConnectionSocketPtr&& socket)函数。
## Network::AcceptedSocketImpl创建
Network::AcceptedSocketImpl类实现了网络编程中socket相关概念的封装，不涉及libevent，类似于前面的Network::TcpListenSocket类。
具体创建逻辑如下：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_connect/1.png)

由图可知，onSocketEvent执行两个操作：
1. 接受连接：执行系统调用accect()
通过关联的Network::TcpListenSocket对象，调用accect()系统调用在listen fd引用的监听流socket上接受一个接入连接。在envoy会创建IoSocketHandleImpl对象保存accect fd，该fd用来和对端(client)socket通信。
2. 响应连接：执行ConnectionHandlerImpl::ActiveTcpListener::onAccept()
创建Network::AcceptedSocketImpl,并关联IoSocketHandleImpl。Network::AcceptedSocketImpl类图如下：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_connect/2.png)

## Server::ActiveTcpSocket创建
Server::ActiveTcpSocket关联ConnectionHandlerImpl::ActiveTcpListener和Network::AcceptedSocketImpl。
具体创建逻辑如下：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_connect/3.png)
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_connect/4.png)
Server::ActiveTcpSocket相关类图如下：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_connect/5.png)
继续深入阅读代码，会调用ConnectionHandlerImpl::ActiveTcpSocket::newConnection()函数，源码如下：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_connect/6.png)
可见最终调用Server::ActiveTcpSocket的成员变量listener_的newConnection函数，listener_为ConnectionHandlerImpl::ActiveTcpListener。
ConnectionHandlerImpl::ActiveTcpListener::newConnection()实现如下：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_connect/7.png)
这里会通过dispatcher_创建Network::ServerConnectionImpl对象。
## Network::ServerConnectionImpl创建
Network::AcceptedSocketImpl类封装了accect socket、Event::Dispatcher(libevent)、以及管理网络过滤器，类似于前面的Network::TcpListenerImpl类。可以理解为是一个中介者，使得Network::AcceptedSocketImpl与事件分发器Event::DispatcherImpl关联起来，从https://ttc.zhiyinlou.com/#/articleDetail?id=3567可以了解到Event::DispatcherImpl是一个事件循环处理器，关联worker和libevent，这样整个流程就可以串起来了。

## accect fd加入libevent
在Network::ServerConnectionImpl构造时，在构造父类Network::ConnectionImpl时,通过Network::AcceptedSocketImpl关联的ioHandle执行文件事件初始化。源码如下：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_connect/8.png)
可见，IoSocketHandleImpl调用成员函数initializeFileEvent()，最终调用libevent库接口event_assign()和event_add(),使libevent event事件关联fd，并在fd上表明我们感兴趣的读写事件，关联读写事件被触发时执行的回调函数。由源代码可知，accect fd上我们对读写事件都感兴趣(Event::FileReadyType::Read | Event::FileReadyType::Write)，回调函数为FileEventImpl实例的mergeInjectedEventsAndRunCb函数，如下图所示：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_connect/9.png)
最终调用的是FileEventImpl成员cb_，cb_为FileReadyCb类型，即std::function<void(uint32_t events)>。回到accect fd加入libevent逻辑最初，我们可以留意到cb_由一个匿名函数对象初始话，匿名函数对象在initializeFileEvent函数中通过Lambda表达式创建。如下图：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_connect/10.png)
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_connect/11.png)
可知，当accect fd有读写事件发生时，读事件会被其他（主动的）socket的写数据所触发，写事件会被响应client时触发，即client/server的write操作，程序会调用 ConnectionImpl::onFileEvent(uint32_t events)，然后判断事件类型执行onWriteReady()或者onReadReady()。


到此为止，我们已经讲解了下游tcp执行connect后，envoy accect连接后，执行accect socket创建和accect fd注册到libevent的操作。accect socket的读写事件响应将在下一篇进行分析。

