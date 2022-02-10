# QUIC-Gateway Design

## Goal

* 无论信令网关还是服务网关（面向移动端的移动网关），非常重要的指标是弱网抗性，本次目标很大程度上是为了提升服务的弱网抗性。网关主要采用QUIC来做，但是针对协议需要有兜底和切换的能力

* 希望有一个扩展性比较好的网关，可以承载部分业务处理（如信令的处理），也可以作为服务的proxy（service proxy）等

* 支持无状态化的server集群

## Features

* 一个标准的server instance（支持start/stop/update/reload等操作），支持cmd和类nginx conf解析

* 多个server可基于规则进行水平扩展（server本身无状态）

* 支持协议的选择（quic/h2/h1.1）。支持长链接

* 支持signal和service的proxy

* 弱网优化

## Archtechure



## Detail



## Extension



