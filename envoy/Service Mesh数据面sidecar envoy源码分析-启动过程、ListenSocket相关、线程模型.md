# Service Mesh数据面sidecar envoy源码分析-启动过程、ListenSocket相关、线程模型
## 一 启动过程
### 1. 入口
source/exe/main.cc中实现了 main() ，是程序运行开始地方，也是代码阅读的入口：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/1.png)
main调用MainCommon::main(int argc, char** argv, PostServerHook hook)静态成员函数，静态函数中会构造Envoy::MainCommon通过智能指针，Envoy::MainCommon会构造MainCommonBase子对象。Envoy::MainCommonBase会构造Server::InstanceImpl子对象通过智能指针管理。类图如下：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/2.png)
Envoy::MainCommon的 run() 方法是envoy运行的主体函数，这个方法调用了 Envoy::MainCommonBase 的 run() ：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/3.png)

![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/4.png)
Envoy::MainCommonBase 的 run() 中调用类 Envoy::Server::InstanceImp 的 run() ：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/5.png)
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/6.png)
到目前为止，程序已经运行起来了。
Envoy::Server::InstanceImp 中实现了envoy的具体功能，接下来就让我们了解一下吧。

## 二 ListenSocket相关
### 1. 创建单例server
#### Envoy::Server::InstanceImpl
类Envoy::Server::InstanceImpl实例化时，初始化了很多私有成员。
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/7.png)
注意构造函数中调用的initialize()函数,这个函数内完成了大量初始化操作，特别注意其中的：
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/8.png)
ListenerManagerImpl对象管理所有的listeners和所有的worker。
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/9.png)
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/10.png)
MainImpl::initialize()函数会创建cluster_manager_，listeners相关包括socket。

到此，Envoy::Server::InstanceImpl实例化完成。下面详细展开listener相关。
### 2. listen socket创建
在MainImpl::initialize()函数中，已经提及到listener的创建，现在我们来详细分析下。
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/11.png)
通过配置信息bootstrap获取listeners信息，然后加入到Envoy::Server::InstanceImpl的ListenerManagerImpl对象中，前面提及到ListenerManagerImpl对象管理所有的listeners和所有的worker。
#### ListenerManagerImpl::addOrUpdateListener()
ListenerManagerImpl::addOrUpdateListener(const envoy::config::listener::v3::Listener& config, const std::string& version_info, bool added_via_api)调用ListenerManagerImpl::addOrUpdateListenerInternal()，addOrUpdateListenerInternal()会创建listener实例。
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/12.png)
#### Server::ListenerImpl
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/13.png)
Server::ListenerImpl关联Server::ListenerManagerImpl。Server::ListenerImpl对应envoy配置信息中的Listener资源。此类中除了包含配置信息还包含程序运行时创建的socket信息，网络编程中服务端的listen fd。socket信息在socket_factory_成员对象中保存，socket_factory_对象并不在Server::ListenerImpl实例化时初始化。ListenerManagerImpl::addOrUpdateListenerInternal()构造Server::ListenerImpl实例后，接着会针对新创建的Server::ListenerImpl实例new_listener进行初始化socket_factory_。
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/14.png)
ListenerManagerImpl::createListenSocketFactory()函数会创建Server::ListenSocketFactoryImpl对象。
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/15.png)
#### Server::ListenSocketFactoryImpl
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/16.png)
首先分析下Server::ListenSocketFactoryImpl实例化时，成员变量的赋值。
由类图可知，Server::ListenSocketFactoryImpl关联Server::ProdListenerComponentFactory，Server::ListenSocketFactoryImpl.factory_由Server::ListenerManagerImpl类实例化，而Server::ListenerManagerImpl由Envoy::Server::InstanceImpl实例化，所以最终Server::ListenSocketFactoryImpl.factory_为Envoy::Server::InstanceImpl.listener_component_factory_的引用。
Server::ListenSocketFactoryImpl.socket_type_由配置信息决定。
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/17.png)
config.core.v3.SocketAddress.Protocol在envoy.yaml配置，未配置默认为TCP。listen socket工厂和socket类型已经被初始化了，接下来通过listen socket工厂去创建listen socket，这里的lsiten socket和网络编程当中的socket模型对应。在ListenSocketFactoryImpl构造函数中，通过调用createListenSocketAndApplyOptions()函数实现。
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/18.png)
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/19.png)
最终，通过Server::ListenSocketFactoryImpl.factory_（ProdListenerComponentFactory对象）调用createListenSocket()来实现。接下来我们看一下真正创建listen socket的函数实现吧：
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/20.png)
可以看到，最终通过socket type创建对应的listen socket实例。
到这里基本明确了，Server::ListenSocketFactoryImpl类中关联Network::TcpListenSocket，Server::ListenerImpl类中关联ListenSocketFactoryImpl。
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/21.png)
Server::ListenerImpl中初始化很多私有成员和构造了网络过滤器链和listener过滤器链，这部分内容本节不做讨论,对应envoy配置信息中的Listener资源,Network::TcpListenSocket是真实的网络编程中的listen socket实现。接下来我们以tcp来分析，分析Network::TcpListenSocket的实现。
#### Network::TcpListenSocket
类图：
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/22.png)
Network::TcpListenSocket是模板类，using TcpListenSocket = NetworkListenSocket<NetworkSocketTrait<Socket::Type::Stream>>;
类模板NetworkListenSocket继承ListenSocketImpl，
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/23.png)
实例化Network::TcpListenSocket时，调用静态Network::ioHandleForAddr函数，最终会调用SocketInterfaceImpl::socket()函数，linux平台下会调用OsSysCallsImpl::socket()函数创建socket，OsSysCallsImpl类封装了系统调用。
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/24.png)
通过SocketInterfaceImpl::makeSocket()函数，构建IoSocketHandleImpl对象，最终的listen fd保存到IoSocketHandleImpl类中。
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/25.png)
由类图可知Network::TcpListenSocket类关联IoSocketHandleImpl类。现在Network::TcpListenSocket实例创建了socket，拥有了listen fd。
到此为止，我们已经分析完listen socket创建逻辑，此时我们需要跳出所有函数调用，回到最初的开始创建，
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/26.png)
我们刚才梳理了第一个listener，别忘记，循环次数是根据envoy配置信息获取的，可能有多个listener需要创建。每次循环结束，Server::ListenerImpl对象就会加入到ListenerManagerImpl的ListenerImpl列表中。
接下来，我们将讨论listen fd何时bind。在“envoy线程模型”章节将讲解listen fd何时listen以及怎么加入到libevent，并由libevent监听是否准备就绪，进而执行io。
### 3 listen socket bind
上一章节，我们讲述了listener的创建，已经生成了listen fd。所有listener由ListenerManagerImpl管理，接下来我们梳理listen fd何时bind。
#### bind listen fd
Network::TcpListenSocket实例化时，会调用setupSocket()函数，
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/27.png)
最终会调用关联的IoSocketHandleImpl对象的bind函数。
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/28.png)

## 三 envoy线程模型
在编程界，基本的线程模型有：1、流水线。2、工作组。
流水线：每个线程反复地在数据系列集上执行同一种操作，并把操作结果传递给下一步骤的其他线程，这就是流水线方式。
工作组：每个线程在自己的数据上执行操作。工作组中的线程可能执行同样的操作，也可能执行不同的操作，但是他们一定独立的执行。
### 1 envoy中的线程模型
envoy采用工作组模式，每个worker从创建后，开始执行各种网络io，包括listener监听的fd io，与下游通信的socket io， 与上游通信的socket io，以及数据集上的操作，比如各种过滤器操作。各个worker各自独立。
envoy中的worker由ListenerManagerImpl管理，包括worker的创建、启动、停止。下面就让给我们来了解一下吧。
### 2 创建worker
Envoy::Server::InstanceImpl实例化时，会构造ListenerManagerImpl对象，Envoy::Server::InstanceImpl关联ListenerManagerImpl。我们来看一下ListenerManagerImpl构造过程。
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/29.png)
ListenerManagerImpl对象管理所有的listeners和所有的worker。
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/30.png)
根据并发度，创建n个worker，保存到std::vector<WorkerPtr> workers_中。
### 3 worker关联Server::ListenerImpl及启动worker
我们回到启动过程章节的最后部分，即Envoy::Server::InstanceImp 的 run()函数阶段。Server::InstanceImpl的run()函数中，可以看到RunHelper对象注册了回调InstanceImpl::startWorkers()。
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/31.png)
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/32.png)
可以看到启动的是ListenerManagerImpl std::vector<WorkerPtr> workers_保存的worker。
启动过程为2层for循环，第一层遍历worker，第二层遍历ListenerManagerImpl 管理的之前创建好的Server::ListenerImpl实例。最终调用addListenerToWorker函数，使worker关联所有Server::ListenerImpl实例。关联后启动worker，此时一个worker启动完成，worker启动在https://ttc.zhiyinlou.com/#/articleDetail?id=3567的第四部分“envoy中的Libevent运行”已经描述，不在赘述，这里主要详细介绍worker怎么关联所有Server::ListenerImpl实例。
#### ListenerManagerImpl::addListenerToWorker()函数
函数内调用void WorkerImpl::addListener(absl::optional<uint64_t> overridden_listener,Network::ListenerConfig& listener, AddListenerCompletion completion)函数，
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/33.png)
最终调用handler_对象的addListener函数。handler_在WorkerImpl构造时提到过，为ConnectionHandlerImpl类。当然这里并不是立马执行，而是通过dispatcher_ post到事件队列中，worker启动后执行。
#### ConnectionHandlerImpl::addListener()函数
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/34.png)
该函数 通过socket类型，创建对应的ActiveListener实例。最后保存ActiveListener实例到worker的handler_对象里。这样worker关联所有ActiveListener间接的关联了所有的Server::ListenerImpl实例。类图如下：
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/35.png)
由类图可知，ActiveListener和TcpListenerImpl双向关联。
#### Network::TcpListenerImpl
TcpListenerImpl类关联Server::ConnectionHandlerImpl::ActiveListener、Network::TcpListenSocket。这里需要注意Network::TcpListenSocket类抽象的是liste socket，socket相关的信息及操作，而Network::TcpListenerImpl可以理解为是一个中介者，使得Network::TcpListenSocket与事件分发器Event::DispatcherImpl关联起来，从https://ttc.zhiyinlou.com/#/articleDetail?id=3567可以了解到Event::DispatcherImpl是一个事件循环处理器，关联worker和libevent，这样整个流程就可以串起来了。Network::TcpListenerImpl会针对Network::TcpListenSocket执行bind和listen相关操作，当然真正的执行者仍是Network::TcpListenSocket中的IoSocketHandleImpl。下面让我们具体了解下listen fd的bind、listen以及怎么加入libevent，进行io。
#### listen fd的监听：listen() 
在构造Network::TcpListenerImpl对象时，TcpListenerImpl构造函数会进行listen fd的bind及listen。如下图：
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/36.png)
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/37.png)

![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/38.png)
可见，IoSocketHandleImpl调用成员函数listen()，最终listen(int fd, int backlog)系统调用将文件描述符fd引用的流socket标记为被动。这个socket后面会被用来接受来自其他（主动的）socket的连接。
#### listen fd加入libevent
在构造Network::TcpListenerImpl对象时，TcpListenerImpl构造函数会进行listen fd的事件注册。如下图：
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/39.png)
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/40.png)
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/41.png)
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/42.png)
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/43.png)
可见，IoSocketHandleImpl调用成员函数initializeFileEvent()，最终调用libevent库接口event_assign()和event_add(),使libevent event事件关联fd，并在fd上表明我们感兴趣的读写事件，关联读写事件被触发时执行的回调函数。由源代码可知，listen fd上我们感兴趣的是读事件(Event::FileReadyType::Read)，回调函数为FileEventImpl实例的mergeInjectedEventsAndRunCb函数，如下图所示：
![image](http://ttc-tal.oss-cn-beijing.aliyuncs.com/1620888861/image.png)
最终调用的是FileEventImpl成员cb_，cb_为FileReadyCb类型，即std::function<void(uint32_t events)>。回到listen fd加入libevent逻辑最初，我们可以留意到cb_由一个匿名函数对象初始话，匿名函数对象在initializeFileEvent函数中通过Lambda表达式创建。如下图：
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/44.png)
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/45.png)
可知，当listen fd有读事件发生时，读事件会被其他（主动的）socket的连接所触发，即client的connect操作，程序会调用TcpListenerImpl::onSocketEvent()，然后调用TcpListenerImpl成员变量cb_,由前面类图可知cb_指向ConnectionHandlerImpl::ActiveTcpListener实例，所以最终调用ConnectionHandlerImpl::ActiveTcpListener::onAccept(Network::ConnectionSocketPtr&& socket)函数。具体逻辑如下：
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/startup_process/46.png)

到此为止，我们需要从函数调用的微观世界跳出来，别忘记，此时我们仅仅梳理了worker关联了一个Server::ListenerImpl的过程，该过程主要构建了Network::TcpListenerImpl实例，该实例可以理解为是一个中介者，使得Network::TcpListenSocket与事件分发器Event::DispatcherImpl关联起来，执行了listen fd的listen及加入libevent中。还记得前面的2层for循环吗？现在只是执行了内层循环的一个listener关联worker，还需要关联n个，内层for循环执行完成后，我们会调用worker.start()函数，启动物理线程，运行事件循环执行器，最终通过libevent去检测我们感兴趣的fd上注册的读写事件是否准备就绪，从而进行io。为什么每个worker会检测所有的listen fd呢？因为这样可以提升并发，我是这样认为的，哈哈