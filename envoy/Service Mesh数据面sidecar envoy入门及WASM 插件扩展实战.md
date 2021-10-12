# 一 envoy介绍
## 1. envoy是什么？
Envoy 是专为大型现代 SOA（面向服务架构）架构设计的 L7 代理和通信总线。该项目源于以下理念：
网络对应用程序来说应该是透明的。当网络和应用程序出现问题时，应该很容易确定问题的根源。Envoy 是一个独立进程，设计为伴随每个应用程序服务运行。
envoy官方文档：https://www.envoyproxy.io/docs
## 2. envoy架构
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/envoy_start/envoy-architecture.png)
envoy网络IO模型：底层采用libevent网络库实现网络IO。每个worker有自己独立的Event loop。

# 二 envoy编译
envoy项目采用bazel项目管理工具进行管理。bazel使用文档：https://docs.bazel.build/versions/master/be/c-cpp.html

1. 在项目根目录下，执行bazel build //source/exe:envoy-static编译envoy项目。
2. 编译是一个很漫长的过程，本人机器配置略低，物理机cpu主频1.6GHz，虚拟机1cpu4核心，4g内存配置，v1.17.0从上午编译到下午6点。Bazel会根据依赖，下载对应的第三方包，有的包需要访问谷歌，比如：com_github_lyft_protoc_gen_star开源库。
  ![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/envoy_start/com_github_lyft_protoc_gen_star-lib.png)
所有依赖包下载成功后，会到源码编译阶段，如下：
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/envoy_start/build_stage.png)
请耐心等待，期间会出现编译失败，按3提及的方式修改即可。编译成功提示如下：
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/envoy_start/build_success.png)
编译成功后，在项目根目录下生成bazel-bin目录，bazel-bin/source/exe/envoy-static为生成的可执行文件。
3.编译问题：项目编译要求比较严格，开启了warning即error的配置，所以有warning 产生编译就会中断，失败。编译大概会遇到2类错误问题：
（1）、error: unused variable 'xxx'，比如，
source/common/config/new_grpc_mux_impl.cc:92:37: error: unused variable 'type_url' [-Werror=unused-variable] for (auto& [type_url, subscription] : subscriptions_) {
修改此类问题比较容易，修改如下：(void)(type_url);后续在编译master分支时，发现官方修改了此问题（修改了部分），方法与我的修改方式相同。
（2）、类型不匹配的赋值操作：定义为相同类型即可。 
# 三 WASM介绍
WebAssembly(缩写为Wasm)是一种用于基于堆栈的虚拟机的二进制指令格式。Wasm被设计为编程语言的可移植编译目标，支持在web上部署客户机和服务器应用程序。可将C、C++、Rust等编译生成WASM格式。
WASM介绍文档：https://webassembly.org/
# 四 HTTP WASM插件编译
1. 下载proxy-wasm-cpp-sdk项目，链接如下：https://github.com/proxy-wasm/proxy-wasm-cpp-sdk/tree/master/example
proxy-wasm-cpp-sdk项目为envoy wasm插件扩展的c++ SDK。
2. 编译http_wasm_example.cc
cd example
执行bazel build http_wasm_example.wasm
编译成功，提示如下：
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/envoy_start/build_success1.png)
bazel-bin/example/下生成http_wasm_example.wasm文件
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/envoy_start/wasm.png)
# 五 插件demo运行
1. envoy运行
- 前期准备：
（1）、envoy执行所需配置文件envoy.yaml。拷贝到/etc下。
（2）、wasm可执行文件envoy_filter_http_wasm_example.wasm
（http_wasm_example.wasm重命名）拷贝到/lib下。
（3）、http server---nginx，用做实际的http服务器，http客户端curl通过envoy访问。
（4）、envoy.yaml配置说明：
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/envoy_start/envoy_config.png)
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/envoy_start/envoy_config1.png)
- 运行envoy：
  在envoy项目根目录下执行./bazel-bin/source/exe/envoy-static -c /etc/envoy.yaml
2. wasm插件demo运行
- 插件的功能:
 （1）、在ResponseBody中添加hello world内容。
 （2）、在ResponseHeaders中添加头信息。
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/envoy_start/alter_responseheaders.png)
- 验证：
 （1）、验证ResponseHeaders中添加头信息
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/envoy_start/verify_alter_responseheaders.png)
 （2）、验证ResponseBody中添加hello world内容
![image](https://github.com/liuqi-sun/Technologyblog/blob/main/envoy/media/envoy_start/verify_alter_responsebody.png)
注：192.168.2.10开发环境ubuntu 16.04 eth0 IP地址。
