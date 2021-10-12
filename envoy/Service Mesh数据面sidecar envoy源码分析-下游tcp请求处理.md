# Service Mesh数据面sidecar envoy源码分析-下游tcp请求处理
上一篇文章主要介绍下游tcp执行connect后，envoy accept连接后，执行accept socket创建、accept fd注册到libevent的详细操作。本文章接着讲述，当下游tcp发起请求时，envoy的具体处理流程。
## 一 下游tcp请求处理概述
当下游tcp在已经建立连接的socket上发起一个请求时，对端socket(服务端的accept socket)会有新的输入，libevent会检测到accept fd准备就绪，进而会进行io，会触发accept fd注册时绑定的回调函数，这个回调函数将是今天我们分析的入口。
在进行具体分析前，首先介绍下处理流程的整体架构。
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/1.png)
由envoy架构图可知，当下游tcp发起请求时，envoy的具体处理流程分为三步：
1. libevent处理
2. filter chains处理
3. 与上游服务交互
libevent处理之前文章已经详细描述了，不了解的朋友可以去查阅。
接下来会介绍filter chains原理及上游服务
### 1 filter chain与network filter
在envoy中，有一个filter chain的概念，什么是filter chain呢？network filter按照有序的顺序链接到一起，形成了一个filter chain，这里又引入了一个概念network filter，什么是network filter呢？network filter是用来处理原始数据的，有三种不同类型的network filter：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/2.png)
network filter详细介绍：						https://www.envoyproxy.io/docs/envoy/v1.17.0/intro/arch_overview/listeners/network_filters	
filter chain详细介绍：
https://www.envoyproxy.io/docs/envoy/v1.17.0/intro/arch_overview/listeners/network_filter_chain
network filter是envoy连接处理的核心，每一个listener有多个filter chains，每个独立的filter chain由一个或者更多的network level (L3/L4) filter组成。 每个独立的filter chain被选择按照他自己的匹配规则，这个动作在创建一个new connection时发生，最终，一个合适的filter chain将被选择。当连接上有事件发生时，network filter被按顺序处理，如果network filter为空，连接将被关闭（默认动作）。
envoy中支持大量的network filter，列表如下：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/3.png)
详细介绍链接：https://www.envoyproxy.io/docs/envoy/v1.17.0/configuration/listeners/network_filters/network_filters#config-network-filters

### 2 上游服务(Upstream)
官方给出如下定义：
Upstream: An upstream host receives connections and requests from Envoy and returns responses.
envoy本身是个代理，所以最终的请求仍然需要真正的服务处理，upstream 就是处理请求的地方。那么问题来了，谁去发起upstream 的连接呢？那就是各个network filter了。

## 二 network filter chain初始化
前面在讲Server::ListenerImpl时，提到该类初始化很多私有成员和构造了网络过滤器链和listener过滤器链。现在我们回到Server::ListenerImpl类来一起梳理下。在构造函数中，发现调用了buildFilterChains()函数，根据名字不难理解是在创建网络过滤器，源码如下：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/4.png)
config_为envoy::config::listener::v3::Listener类型，和配置文件中的Listener相对应，这些类都是程序自动生成的。过滤器链由过滤器管理器管理，ListenerImpl关联过滤器管理器。Server::ListenerImpl对应envoy配置信息中的Listener资源。最终过滤器链通过ListenerFilterChainFactoryBuilder创建，在创建每一个过滤器链的时候，会循环遍历过滤器链中的所有网络过滤器，最终根据proto_config类型创建每一个网络过滤器对应的网络过滤器工厂函数对象，以HttpConnectionManager网络过滤器为例：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/5.png)
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/6.png)
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/7.png)

最终FilterChainImpl类成员变量filters_factory_保存所有网络过滤器工厂函数对象。
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/8.png)

## 三 ConnectionHandlerImpl::ActiveTcpConnection创建并关联过滤器链
### 1 ActiveTcpConnection创建
一个ActiveTcpConnection的创建代表着有一个活跃的tcp连接与下游tcp通信。
还记得Network::ServerConnectionImpl的创建过程吗？在“Service Mesh数据面sidecar envoy源码分析-下游tcp连接处理”一文中，讲解了Network::ServerConnectionImpl的创建。ServerConnectionImpl可以理解为是一个中介者，使得Network::AcceptedSocketImpl与事件分发器Event::DispatcherImpl关联起来。ActiveTcpConnection也是在这个时候创建的，它关联了ServerConnectionImpl。
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/9.png)

### 2 ActiveTcpConnection关联过滤器链
当ActiveTcpConnection关联了过滤器，后续请求来临时，关联的过滤器都会被处理。这样每个ActiveTcpConnection都会关联与自己匹配的过滤器链，当连接上有事件发生时，network filter被按顺序处理。而ListenerImpl::createNetworkFilterChain()就是完成了这个功能的函数。
前面在讲解ConnectionHandlerImpl::ActiveTcpListener时讲到，worker关联Server::ListenerImpl是通过关联所有ActiveTcpListener间接的关联了所有的Server::ListenerImpl实例，可知Server::ListenerImpl是ConnectionHandlerImpl::ActiveTcpListener子对象config_。由代码可知，我们需要获取过滤器链工厂进而关联网络过滤器链，
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/10.png)
可知config_的过滤器链工厂为自己。所以我们直接查看ListenerImpl::createNetworkFilterChain(Network::Connection& connection,const std::vector<Network::FilterFactoryCb>& filter_factories)函数，
参数connection：
Network::ServerConnectionImpl实例。
参数filter_factories
network filter chain初始化时，FilterChainImpl类成员变量filters_factory_注册的网络过滤器工厂函数对象。

下面我们以HttpConnectionManager网络过滤器工厂函数对象为例，介绍HttpConnectionManager过滤器是怎么注册到ActiveTcpConnection的。
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/11.png)
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/12.png)
函数对象factory，调用()运算符，参数为filter_manager，filter_manager就是ServerConnectionImpl。
接下来执行ServerConnectionImpl的addReadFilter函数，注册读网络过滤器HttpConnectionManager。
最终，由ServerConnectionImpl父类ConnectionImpl中的FilterManagerImpl类来管理网络过滤器，存储到list数据结构中--std::list<ActiveReadFilterPtr> upstream_filters_。
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/13.png)
当所有网络过滤器都注册完成后，ServerConnectionImpl会进行initializeReadFilters()初始化工作，最终调用FilterManagerImpl的initializeReadFilters()函数，
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/14.png)
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/15.png)
在onContinueReading()函数中，我们看到会进行遍历已经注册的网络过滤器链，分别进行onNewConnection和onData事件函数，目前ActiveTcpConnection创建是在tcp连接阶段触发的回调函数中创建的，所以read_buffer没有内容，此逻辑不会执行。

总之，本人认为ConnectionHandlerImpl::ActiveTcpConnection创建并关联过滤器链，是一个初始化的阶段，使ActiveTcpConnection对象准备就绪，方便后续有请求到来时执行响应的操作。

## 四 下游tcp请求处理
前面的工作都是为了这一章而准备的。下面我们就来一起梳理当下游请求到来时，envoy做了什么，咱们还是会以http网络过滤器为例展开。
首先我们应该回到accept fd读事件触发时，执行的回调函数。在“Service Mesh数据面sidecar envoy源码分析-下游tcp连接处理”一文中，已经详细介绍了下游tcp执行connect后，envoy accect连接后，执行accect socket创建、accect fd注册到libevent的详细操作，在“accect fd加入libevent”小节中，详细描述了读写事件发生时，程序会调用 ConnectionImpl::onFileEvent(uint32_t events)，然后判断事件类型执行onWriteReady()或者onReadReady()。接下来，让我们详细分析下ConnectionImpl::onFileEvent逻辑。源码如下：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/16.png)
可知，根据不同的事件，执行不同的逻辑，这一节中咱们讨论的是下游tcp请求处理，所以会触发读事件，会执行onReadReady()方法。
### 1 读取accept数据
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/17.png)
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/18.png)
通过transport_socket_读取socket数据，transport_socket_最终通过callbacks_对象读取socket，callbacks_其实就是ConnectionImpl，在ConnectionImpl对象构造时关联。
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/19.png)

### 2 network filter被按顺序处理
获取到数据后，开始执行连接关联的网络过滤器。关联过程在前面准备工作已经讲述，忘记的可以返回去查看。
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/20.png)
filter_manager_对象还记的吧，是用来管理活跃tcp连接关联的网络过滤器的。
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/21.png)
此时，执行的是onData逻辑，目前是请求操作，我们已经读到了请求数据。我们时刻要明确进程目前处于什么状态，否则会一团糟的。

### 3 网络过滤器处理数据
我们以http网络过滤器为例，来详细讲述这个阶段。
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/22.png)
由源码可知，Http::ConnectionManagerImpl为注册的http过滤器。
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/23.png)

由源码可知，http网络过滤器处理数据过程中，会分为两步：创建http编解码器和数据处理。
#### 3.1 http编解码器创建
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/24.png)
根据envoy.yaml配置文件的编解码器类型，创建不同的编解码器，关联Network::ConnectionImpl。对于http1，最终会创建Http::Http1::ServerConnectionImpl编解码器。
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/25.png)
#### 3.2 http编解码器处理http头数据
Http::Http1::ServerConnectionImpl编解码器最总把解码工作交给http_parser，http_parser是一个nodejs_http_parser c库，利用状态机机制去执行工作，起始状态为s_start_req，每一种状态对应的执行函数在settings_中注册。主要讲解下ConnectionImpl::onMessageBeginBase()与ConnectionImpl::onMessageCompleteBase()的工作，分别对应消息开始和消息完成阶段。源码如下：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/26.png)
ConnectionImpl::onMessageBeginBase()为第一个状态对应的执行体，ConnectionImpl::onMessageCompleteBase()为最后一个状态对应的执行体。
##### 3.2.1 创建Http::Http1::ServerConnectionImpl::ActiveRequest
在ConnectionImpl::onMessageBeginBase()中会创建ActiveRequest，ActiveRequest是一个活跃的http请求，该类关联了请求解码器和响应编码器。请求解码器指向ConnectionManagerImpl::ActiveStream。响应编码器为ResponseEncoderImpl。
##### 3.2.2 创建ConnectionManagerImpl::ActiveStream
在ConnectionImpl::onMessageBeginBase()中就会创建ActiveStream，Http::Http1::ServerConnectionImpl会通过http网络过滤器创建ActiveStream并会管理(ConnectionManagerImpl)ActiveStream，存储在list中。ActiveStream在一个连接之上创建，编解码http头和数据，通过FilterManager管理http网络过滤器下的http过滤器。这里引入了http过滤器，后续会详细讨论。
##### 3.2.3 http过滤器
像网络过滤器一样，envoy支持http级别的过滤器栈，过滤器可以操作http级别的消息，按照顺序去迭代，在迭代过程中可以停止或者继续随后的过滤器。有三种http级别的过滤器：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/27.png)
详细介绍文档链接：https://www.envoyproxy.io/docs/envoy/v1.17.0/intro/arch_overview/http/http_filters
##### 3.2.4 通过ActiveStream中的FilterManager管理http过滤器
首先，我们了解下http过滤器的初始化阶段，在第二章network filter chain初始化中，讲述了网络过滤器工厂函数对象的创建过程，这些函数对象在ActiveTcpConnection的创建时，会把对应的网络过滤器注册到这个活跃的tcp连接中。再仔细阅读http网络过滤器工厂函数对象的创建过程这段逻辑时，我们会发现，这里进行了HttpConnectionManagerConfig的构造，HttpConnectionManagerConfig是http网络过滤器相关配置的实现，这里面会进行http过滤器的初始化。
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/28.png)
HttpConnectionManagerConfig构造函数执行如下逻辑：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/29.png)
循环处理envoy.yaml中配置的所有http过滤器。
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/30.png)
通过proto_config的typed_config获取http过滤器工厂，比如我们获取到本地限流(local rate limit)过滤器工厂(LocalRateLimitFilterConfig)，然后创建http过滤器工厂函数对象(Http::FilterFactoryCb).
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/31.png)

http WASM过滤器工厂，![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/56.png)
所有配置的http过滤器，最终会在FilterFactoriesList filter_factories_进行保存，filter_factories_中管理的是StaticFilterConfigProviderImpl对象，StaticFilterConfigProviderImpl关联http过滤器工厂函数对象和过滤器name
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/32.png)。
HttpConnectionManagerConfig实例，在http网络过滤器实例化时被关联。
接下来，我们了解下http过滤器是如何被ActiveStream中的FilterManager管理的。
在http_parser最后一个阶段中，请求解码器ConnectionManagerImpl::ActiveStream在解码http头的过程中，会通过遍历FilterFactoriesList filter_factories_将http过滤器注册到ActiveStream中，由FilterManager管理。
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/33.png)
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/34.png)
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/35.png)
由代码可知，最终会调用函数对象的()运算符，callbacks为FilterManager。如下为限流过滤器被FilterManager管理实现：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/36.png)
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/37.png)
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/38.png)

最终，http过滤器被注册到ActiveStream中，由FilterManager管理，过滤器被注册到流解码和编码过滤器列表中。

##### 3.2.5 http过滤器执行
在http_parser最后一个阶段中，注册所有http过滤器后，FilterManager会进行http头解码。
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/39.png)
http限流过滤器执行：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/40.png)
http WASM过滤器执行：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/41.png)
http route过滤器执行：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/42.png)
envoy运行信息打印：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/43.png)
##### 3.2.6 http router过滤器解码头
router过滤器比较重要，会创建与upstream服务通信的socket。接下来我们来梳理下这部分内容。
首先会创建连接池，更具envoy.yaml配置信息选择不通的工厂去创建。我的clusters配置中没有配置upstream_config信息。选择"envoy.filters.connection_pools.http.generic"对应的工厂。
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/44.png)
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/45.png)
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/46.png)
最终会创建FixedHttpConnPoolImpl连接池。
接下来会创建新连接ActiveClient，连接upstream。具体实现如下：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/47.png)
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/48.png)
通过FixedHttpConnPoolImpl连接池来实例化ActiveClient。通过调用client_fn_函数来创建，client_fn_在FixedHttpConnPoolImpl构造时初始化。代码如下：
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/49.png)
由构造函数实现可知，第8个实参初始化client_fn。
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/50.png)
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/51.png)
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/52.png)
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/53.png)
最终，创建Network::ClientConnectionImpl，将socket加入libevent。
最后通过FixedHttpConnPoolImpl创建CodecClientProd。
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/54.png)
由FixedHttpConnPoolImpl构造函数实现可知，第9个实参初始化codec_fn_。
最终CodecClientProd调用ClientConnectionImpl::connect()连接upstream。
![image.png](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/downstream_tcp_request/55.png)


到此为止，http编解码器已经处理完http头数据，通过router过滤器并与upstream建立了连接。由于这一块逻辑太多且复杂，加上时间有限，很多地方只是按功能进行了分析，代码没有特别详细的分析，类与类之间的关系也没有详细梳理，感兴趣的朋友可以自己仔细研读。body数据处理及upstream响应处理后续有时间再梳理。

文章中理解不正确、不到位的地方希望大家指出来，咱们一起讨论学习，一起进步。


未完待续。