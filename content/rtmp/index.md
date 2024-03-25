+++
title = "rtmp 协议解析"
date = 2024-01-24

[taxonomies]
tags = ["rtmp", "流媒体协议"]
+++

关于 rtmp 协议的解析思路。

<!-- more -->

# 握手

要建立一个有效的 rtmp 连接，首先要握手，类似于 tcp 的 3 次握手。客户端要向服务器发送 c0,c1,c2 三个数据包,服务器向客户端发送 s0,s1,s2 三个数据包，然后才会开始有效的数据传输。rtmp 并没有规定这几个数据包统一的接受和发送顺序，只是保留了一定的约束。

- c2 必须在收到 s1 后发送
- 必须收到 s2 后，才能发送其他信息
- 在收到 c0 之后才能发送 s1
- 在收到 c1 之后才能发送 s2
- 在收到 c2 之后才能发送其他信息

> 实际实现中，c0c1,s0s1 往往同时发出

c0/s0
一个字节，用于协商 rtmp

c1/c1
1536 个字节，可以用来检验，也可以什么都不做，实行检测是复杂握手，不检验为简单握手。

复杂握手
将 1536 个字节的后 1528 个字节平等分为 2 部分，一部分存储 public key，一部分存储 digst，但是并没有字段声明那个是 digst，需要分别验证，

综上所述，rtmp 握手时要分布检测是否为复杂握手和简单握手。

# 数据传输

rtmp 中存储数据的基础单元是 message，但是传输数据的基础单元是 chunk。一个 message 会对应一个或者多个 chunk。所以 chunk 中有着 message 的唯一标识用来区分 chunk 的所属。

chunk 的实际定义如下:

- chunk base header
- chunk message header
- extended time stamp
- chunk data

chunk base header 是不定长的。它分为两部分，由第一个字节的前两个 bit 决定总体的长度。
