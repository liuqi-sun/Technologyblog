# Service Mesh数据面sidecar envoy源码分析-网络IO

## 一 说明
该系列所有笔记主要分析envoy 底层网络IO模型、线程模型、网络过滤器、http过滤器、http 请求处理流程分析。http请求处理流程涉及的模块较多，可能需要按子模块详细分析。
这里阅读的是envoy 1.17.0版本的代码，目的是对envoy的代码结构、启动过程、主要模块建立基本的认知，以后遇到“感觉比较模糊”的内容时，可以快速通过阅读源码求证。
## 二 Libevent
libevent是一个用C语言编写的、轻量级的开源高性能事件通知库(an event notification library),libevent API提供了一种机制，当在文件描述符上发生特定事件或达到超时时，执行回调函数。目前, libevent支持 /dev/poll, kqueue(2), event ports, POSIX select(2), Windows select(), poll(2), and epoll(4). 详细介绍参考https://libevent.org/。envoy底层网络IO采用libevent库实现，每个worker有自己独立的Event loop。
## 三 envoy中的libevent
### Dispatcher 和 Libevent
在讨论libevent前，有必要先讲解下worker，在envoy中worker用来处理网络请求等，每一个worker有独立的事件循环，在envoy中事件循环用Dispatcher 实现，Dispatcher 封装libevent来进行网络IO。
### worker
这里不详细讨论worker的启动，后续文章会分析。这里着重介绍下worker的创建及与Dispatcher 和 Libevent的关系，通过介绍worker和Dispatcher 和 Libevent的关系可以深入的理解libevent在envoy中是如何使用的。
worker通过worker工厂类来创建，实现如下：
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/network_io/1.png)
通过程序启动worker的并发度个数n，创建n个worker。
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/network_io/2.png)
可以看到，WorkerImpl类包含Event::DispatcherImpl和ConnectionHandlerImpl成员对象。
### Event::DispatcherImpl
通过查看Event::DispatcherImpl类定义，可以看到包含LibeventScheduler base_scheduler_;成员对象，
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/network_io/3.png)
### Event::LibeventScheduler
通过查看Event::LibeventScheduler类定义，可以看到包含Libevent::BasePtr libevent_;成员对象，
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/network_io/4.png)
Libevent::BasePtr实际上是一个模板类CSmartPtr<event_base, event_base_free>;
libevent_对象初始化，如下：
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/network_io/5.png)
这里已经看到了调用的libevent接口event_base_new()。
### Event::CSmartPtr<event_base, event_base_free>
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/network_io/6.png)
Libevent是一个C库，而Envoy是C++，为了避免手动管理这些C结构的内存，Envoy通过继承unique_ptr的方式重新封装了这些libevent暴露出来的C结构。通过unique_ptr智能指针管理的event_base*指针和释放内存函数通过参数传给unique_ptr。
### 类图
为了更清晰的展示WorkerImpl、Event::DispatcherImpl、Event::LibeventScheduler、Event::CSmartPtr<event_base, event_base_free>之间的关系，给出如下类图。
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/network_io/7.png)

## 四 envoy中的Libevent运行
libevent的运行是通过worker间接启动的。通过ListenerManagerImpl::startWorkers启动所有worker，这里不详细介绍startWorkers流程，引入此函数只是为了更好的描述libevent执行流程，让读者更容易理解。
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/network_io/8.png)
worker调用成员函数start类图中给出，start函数通过线程工厂类，创建物理线程，最终linux平台通过pthread_create函数创建物理线程。感兴趣的读者可以自行深入阅读。
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/network_io/9.png)
最终，线程入口函数threadRoutine被执行，此函数即worker的成员函数，类图中已给出，threadRoutine会调用dispatcher_->run(Event::Dispatcher::RunType::Block);Event::DispatcherImpl成员函数run，
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/network_io/10.png)
DispatcherImpl::run(RunType type)会调用base_scheduler_.run(type);Event::LibeventScheduler成员函数，
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/network_io/11.png)
LibeventScheduler::run(Dispatcher::RunType mode)最终会调用libevent库中的event_base_loop(libevent_.get(), flag);函数。
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/network_io/12.png)
至此，整个网络IO可以通过libevent来进行处理，worker通过调用关联的Event::DispatcherImpl成员对象的函数run，Event::DispatcherImpl通过调用关联的Event::LibeventScheduler对象的run函数，Event::LibeventScheduler最终调用libevent接口event_base_loop，event_base_loop依赖Event::LibeventScheduler的子对象Event::CSmartPtr<event_base, event_base_free>，最终worker的工作交给libevent来执行。
libevent封装了多路复用io select poll系统函数，以及linux专属的epoll函数，当然还有其他操作系统的io处理函数。libevent会监听注册的fd是否准备就绪，然后进行io。当然整个流程未分析fd注册到libevent相关逻辑，此部分逻辑在后续文章会进行梳理。