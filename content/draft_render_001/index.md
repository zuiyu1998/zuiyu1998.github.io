+++
title = "论draft-render中顶点绘制流程"
date = 2025-05-19

[taxonomies]
tags = ["frame_graph", "renderer", "draft-render", "fyrox-render"]
+++

论draft-render中顶点绘制流程
<!-- more -->

# 正常的流程
- 获取对应的顶点数据和索引数据，直接在pass上调用。

定义一个对象包括顶点数据和索引数据，通过这个对象执行此流程。

