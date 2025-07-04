+++
title = "opemim-msg-transfer"
date = 2025-04-09

[taxonomies]
tags = ["openim", "redis", "mongo"]
+++

这是 openim 源码解析系列的一篇文章。

<!-- more -->

# msg-transfer

这是整个 im 系统中靠后的位置。在 openim 的系统架构中，msg-transfer 获取 kafka 中的数据，对每个消息生成唯一 id，并在数据库中进行归档存储。

方案介绍:

- 每个会话的消息 id 是唯一的。消息存储是以会话作为基础存储的，采用的读扩散(?)
