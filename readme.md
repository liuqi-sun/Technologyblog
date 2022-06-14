- [api-gateway](#api-gateway)
  - [Overview](#overview)
  - [Build](#build)
  - [Run](#run)
  - [Build docker image](#build-docker-image)
  - [Deploy to k8s](#deploy-to-k8s)
  - [Test module](#test-module)
# api-gateway
## Overview
架构图  
![architecture](istio-agent/doc/picture/api-gateway架构图.png)  

api-gateway有两个主要模块：  
  - xds 客户端  
  主要与istio xds服务端通信，进行xds资源的订阅，基于grpc-go xds实现，grpc支持SotW ADS  
  - nginx控制器  
    - 管理nginx进程，包括启动，关闭  
    - xds资源类型数据进行解析、处理，生成nginx server、upstream、loction元数据，通过go tmpl技术生成nginx.conf文件
    - 通过lua动态实现对upstream的访问及灰度
## Build
```sh
cd api-gateway/istio-agent/cmd  
go build -o api-gateway ./main.go  
```
## Run
```sh
cd api-gateway/istio-agent/cmd  
./api-gateway
```
## Build docker image
```sh
cd api-gateway/istio-agent/docker  
```
复制api-gateway到docker目录，执行  
```sh
make docker_api_gateway
```
## Deploy to k8s
```sh
cd api-gateway/istio-agent/deploy  
kubectl apply -f ./api-gateway.yaml
```
## Test module
### Canary  
- header灰度测试  
以bookinf为例进行测试，在访问reviews服务时，如果携带“end-user:jason”头，请求会被灰度到reviews v2版本，v2版本会调用ratings服务，并使用1到5个黑色星形图标来显示评分信息，
否则，请求会被灰度到reviews v3版本，v3 版本会调用ratings服务，并使用1到5个红色星形图标来显示评分信息。
```sh
curl  http://192.168.49.2:30339/reviews/0 -H "end-user:jason" -H "Host:reviews" -i
HTTP/1.1 200 OK
Server: nginx/1.20.2
Date: Tue, 14 Jun 2022 08:51:30 GMT
Content-Type: application/json
Content-Length: 379
Connection: keep-alive
x-powered-by: Servlet/3.1
content-language: en-US
x-envoy-upstream-service-time: 1353
x-envoy-decorator-operation: reviews.default.svc.cluster.local:9080/*

{"id": "0","reviews": [{  "reviewer": "Reviewer1",  "text": "An extremely entertaining play by Shakespeare. The slapstick humour is refreshing!", "rating": {"stars": 5, "color": "black"}},{  "reviewer": "Reviewer2",  "text": "Absolutely fun and entertaining. The play lacks thematic depth when compared to other plays by Shakespeare.", "rating": {"stars": 4, "color": "black"}}]}
```
```sh
curl  http://192.168.49.2:30339/reviews/0  -H "Host:reviews" -i
HTTP/1.1 200 OK
Server: nginx/1.20.2
Date: Tue, 14 Jun 2022 09:21:41 GMT
Content-Type: application/json
Content-Length: 375
Connection: keep-alive
x-powered-by: Servlet/3.1
content-language: en-US
x-envoy-upstream-service-time: 639
x-envoy-decorator-operation: reviews.default.svc.cluster.local:9080/*

{"id": "0","reviews": [{  "reviewer": "Reviewer1",  "text": "An extremely entertaining play by Shakespeare. The slapstick humour is refreshing!", "rating": {"stars": 5, "color": "red"}},{  "reviewer": "Reviewer2",  "text": "Absolutely fun and entertaining. The play lacks thematic depth when compared to other plays by Shakespeare.", "rating": {"stars": 4, "color": "red"}}]}
```

192.168.49.2为minikube的ip，30339为api-gateway的nodeport。可以通过"rating": {"stars": 5, "color": "red"}中的color字段的值来判定处理请求的reviews服务为哪个版本。


