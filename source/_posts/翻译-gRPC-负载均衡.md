title: '[翻译] gRPC 负载均衡'
author: Haoxiang Zhao
date: 2018-11-12 18:58:26
tags:
---
# 前言
这是gRPC 官方博客的文章, 由 makdharma 大佬发布于2017.6.15, 原文见[gRPC Load Balancing](https://grpc.io/blog/loadbalancing).最近在看gRPC, 看到负载,服务发现这块, 顺路搜了一下就发现了这篇, 内容不错, 翻译学习加留存.
# 正文
无论出于什么目的, 一个大规模的gRPC部署中, 一个服务一般都会有若干个实例, 同时还有着大量的调用端.但是单个server实例, 其运算负载能力是有限的. 负载均衡就是用来高效地将来自客户端的请求分发到可用的服务端, 以达到负载能力的提升.
## 为什么使用gRPC?
gRPC基于[HTTP/2](https://developers.google.com/web/fundamentals/performance/http2/?hl=zh-cn)构建.
(译: [gRPC Design and Implementation](https://platformlab.stanford.edu/Seminar%20Talks/gRPC.pdf)有介绍为啥选用http2). HTTP/2属于应用层(L7), 运行在传输层(TCP, L4)之上, 当然更在网络层(IP, L3)之上.(译: [网络分层](https://en.wikipedia.org/wiki/Network_layer)). 沾了HTTP/2协议的光, gRPC有着许多传统基于HTTP/REST/JSON通讯不具备的[优势](https://http2.github.io/faq/#why-is-http2-binary), 比如
1. 二进制通讯(来自HTTP/2)
2. 多路复用(来自HTTP/2)
3. 请求/返回头压缩(来自HTTP/2)
4. 强类型定义, 便于约定服务和消息体(来自[Protobuf](https://developers.google.com/protocol-buffers/))
5. 多语言支持
此外, gRPC还能与诸如服务发现, 名字解析, 负载均衡, 路由追踪和监控等生态组件无缝整合.
## 负载均衡策略
### 代理模式还是客户端?
**注:**代理模式的负载, 在某些场合也称为服务端负载均衡.
使用代理模式还是客户端负载均衡是架构层面一个主要的问题. 在基于代理的负载均衡模式中, 客户端将RPC调用发给负载均衡器(LB). LB再负责将请求发往可用的目标server. LB负责监控后端server的存活并实现一个请求分发算法. 客户端对实际server无感知.这也允许客户端可以是不受信的. 这种架构, 通常用于服务调用来自公网的面向客户的服务.如下图所示, 客户端将请求发到LB(#1).LB将流量发往对应的后端server(#2). Server日常返回自己的状态给LB(#3, 译: 这个状态反馈,可以是主动的, 可以是被动的,由具体实现定)
![proxy](/images/翻译-gRPC-负载均衡/proxy.png)
客户端负载模式下, 客户端需要知道到底有多少server可用, 然后每次RPC时再选取一个实际调用.客户端负责记录server的状态, 并实现负载均衡的算法. 比较简单常见的就是[round-robin](https://en.wikipedia.org/wiki/Round-robin_scheduling)轮巡, 也不用考虑server实际能力, 简单的下一个就好了.如下图所示, 客户端将请求发往特定的后端(#1). 通常, server会将自身状态和RPC的结果一并返给client. 然后client更新server的状态, 以便下次负载计算.
![client](/images/翻译-gRPC-负载均衡/client.png)
这两种模式各有利弊

|      |   服务端模式                                                                 | 客户端模式                                                                                                                                          |
|------|----------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|
| 优<br/>点 | 1. 简化客户端开发<br/>2. 客户端对服务端无感知<br/>3. 可以为不受信的客户端提供服务  | * 性能高(没有额外的转发消耗)                                                                                                                        |
| 缺<br/>点 | 1. LB增加了数据传输路程<br/>2. 延迟增加v3. LB的吞吐量可能限制服务扩展伸缩 | 1. 客户端变得复杂<br/>2. 客户端需要自己处理server的状态<br/>3. 自己实现均衡算法<br/>4. 多语言和维护成本增加<br/>5. 客户端需要是受信的, 或者客户端侧的LB处理受信问题 |

### 服务端负载策略
服务端的负载, 可以是[传输层](https://www.nginx.com/resources/glossary/layer-4-load-balancing/)的(L3/L4), 也可以是应用层的(L7). 
传输层负载,LB直接关闭一个TCP连接, 再与新选中的serveri建立连接就好了.至于应用层的数据(HTTP/2和gRPC帧), LB就从与client的连接简单的拷贝到与server的连接.L3/L4层的LB只做很少的处理, 相较L7而言, 延迟会更小, 消耗资源会更少.
应用层(L7)的负载均衡, LB负责接收和h处理HTTP/2请求. LB同时可以根据请求的内容决定u分发到哪个server.比如http请求header中的[session cookie](https://en.wikipedia.org/wiki/HTTP_cookie#Session_cookie).
一旦LB选定了后端server, 就与其建立一个HTTP/2连接,然后转发由客户端发来的请求.
这两种策略, 也是各有利弊, 不同场合,不同选择.

| 场合                       | 推荐                    |
|----------------------------|-------------------------|
| RPC server经常变动         | L7                      |
| 存储或者计算亲和性很重要时 | L7结合类似cookies的方法 |
| 资源消耗最小               | L3/L4                   |
| 延迟至关重要时             | L3/L4                   |

### 客户端负载策略
#### 胖客户端
**胖客户端**, 就如名字一样形象, 负载策略,server信息全交由client处理.这样的客户端一般还会通过类库整合服务发现, 名字解析, 配置中心. 因为什么都有, 也因此得名:胖.
#### 外部负载均衡器
与server端负载类似, 存在一个外部的LBserver, 但是它只负责"指路", 不负责"带路". 客户端通过查询这个特殊的LB拿到最佳的server去调用. 记录server状态, 实现LB算法这种繁重的工作都放在均衡器来处理. 不过在有外置LB的情况下, 客户端侧还是能自己实现一些简单的负载算法. gRPC就是使用了这种模型. 具体细节见[文档](https://github.com/grpc/grpc/blob/master/doc/load-balancing.md).

下图展示的就是这个模型. 客户端从LB拿到至少一个server地址(#1). 然后自己根据地址直接向server发起RPC请求(#2). server日常将自己的状态上报给LB(#3).LB自己在与其他诸如名字解析服务发现等组件通讯(#4).
![clinet-lookaside](/images/翻译-gRPC-负载均衡/client2.png)

## 最佳实践
根据具体的部署情况, 我们建议

| 场合和现有部署形式                                                            | 推荐                                                                                                                                                                                                         |
|-------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 客户端和服务端之间流量很大<br/>客户端是受信的                                 | 胖客户端<br/>客户端侧负载,比如ZK/etcd/consul/Eureka                                                                                                                                                          |
| 传统部署形式, 反向代理<br/>客户端和服务端之间需要受信连接                     | 代理模式<br/>如果在用GCP, GCLB提供L3/L4负载<br/>haproxy(L3/L4)<br/>[nginx 1.13.10](https://www.nginx.com/blog/nginx-1-13-10-grpc/)<br/>如果需要session stickiness, 使用[Envoy](https://www.envoyproxy.io/)做L7代理 |
| 微服务架构<br/>高性能(低延迟, 大流量)<br/>客户端可以不受信                      | 外置负载均衡器<br/>基于gRPC-LB协议的客户端LB                                                                                                                                                                 |
| 使用服务网格比如[Linkerd](https://linkerd.io/), [Istio](https://istio.io/zh/) | 服务网格<br/>使用[Istio](https://istio.io/zh/)内置的LB或者[Envoy](https://www.envoyproxy.io/)                                                                                                                |