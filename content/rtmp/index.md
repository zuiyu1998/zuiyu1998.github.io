+++
title = "rtmp 协议解析"
date = 2024-01-24

[taxonomies]
tags = ["rtmp", "流媒体协议"]
+++

关于 rtmp 协议的解析思路。

<!-- more -->

# 握手

# 数据传输

rtmp 中存储数据的基础单元是 message，但是传输数据的基础单元是 chunk。一个 message 会对应一个或者多个 chunk。所以 chunk 中有着 message 的唯一标识用来区分 chunk 的所属。

chunk 的实际定义如下:

- chunk base header
- chunk message header
- extended time stamp
- chunk data

chunk base header 是不定长的。它分为两部分，由第一个字节的前两个 bit 决定总体的长度。
