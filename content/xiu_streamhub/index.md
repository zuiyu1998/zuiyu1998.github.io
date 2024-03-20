+++
title = "论xiu streamhub的设计"
date = 2024-01-24

[taxonomies]
tags = ["xiu", "流媒体服务器"]
+++

xiu streamhub 的设计。

<!-- more -->

# 需求

可以任意的添加多个协议流，对每个协议流都可以进行转发。这是一个一对多，一对多的关系。
并且这种关系不是默认存在的，而是一步步添加的。

观察者模式，这是十分符合这一抽象的。每个客户端不需要知道这个流来自那个协议，而是只需要获取数据的流。

# 主要设计

StreamHubEventSender

- 负责订阅和发布，类似 Observer

Transmitter

- 每一个 publish 的实际管理者，负责数据流的管理

DataSender/DataReceiver

- 每个客户端实际获取的数据流
