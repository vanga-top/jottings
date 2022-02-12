# QUIC-Gateway Design

## Goal

* 无论信令网关还是服务网关（面向移动端的移动网关），非常重要的指标是弱网抗性，本次目标很大程度上是为了提升服务的弱网抗性。网关主要采用QUIC来做，但是针对协议需要有兜底和切换的能力

* 希望有一个扩展性比较好的网关，可以承载部分业务处理（如信令的处理），也可以作为服务的proxy（service proxy）等

* 支持无状态化的server集群

## Features

* 一个标准的server instance（支持start/stop/update/reload等操作），支持cmd和类nginx conf解析 （内部采用module方式进行管理）

* 多个server可基于规则进行水平扩展（server本身无状态）

* 支持协议的选择（quic/h2/h1.1）

* 支持http端连接和长连接/rpc service的网关proxy （采用plugin的方式进行扩展）

* high preformance is very important


## Archtechure

server instance hold modules and it can install third part plugins 

![](./assets/imgs/gateway-server-arch.png)

这里的代码实现我会参考caddy的部分设计，原因很简单，不想重复造轮子。但是caddy的设计我觉得并不是太好。从1.0到2.x的版本，把模块化管理做的更清晰了，但最核心部分的层次结构没有改，这个比较遗憾。导致了代码的结构并不能很顺畅的表达运行的结构。

## Detail



## Extension



