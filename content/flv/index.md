+++
title = "flv"
date = 2024-01-24

[taxonomies]
tags = ["flv"]
+++

flv 协议。

<!-- more -->

# 构成

flv 协议的构成如下:

- FLV header
- Flv body
- PreviousTagSize0
- tag
  - tag header
  - tag body

# Flv header

声明 flv 的版本，音视频信息。前 3 个字节为 flv，第 4 个字节表示版本，第 5 个字节表示音视频信息。第 6 个到第 9 个字节为 header 的大小。

# flv body

由连续的 PreviousTagSize0 和 tag 组成。

# tag

tag 分为 header 和 body。header 分为如下字段：

- tag_type,表示当前 tag 实际保存的数据。
- data_size，表示当前 tag body 的大小。
- timestamp，时间戳
- timestamp_extended，时间戳
- streamID,
- data，body 数据

# script tag

script tag 的数据为一个 amf 格式的对象数据。通常情况下它分为两部分，第一部分为 amf 格式的字符串名字叫"onMetaData"，另一部分为 amf 的数组，返回一些定义的数据。

# video tag

video data 中第 0-4bit 声明了 video 的类型，第 4-8bit 声明了 video 的 code 类型，剩余都是 video 数据。

# audio tag

audio tag 中第 0-4bit 声明了 audio 的 format。第 4-6 个 bit 声明了 audio 的 rate,第 6 个 bit 声明了 size，第 7 个 bit 声明了声道，剩下的是数据。
