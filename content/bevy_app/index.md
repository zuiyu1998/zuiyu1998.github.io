+++
title = "从零开始bevy-app"
date = 2025-10-20

[taxonomies]
tags = ["bevy", "app"]
+++

这是探究 bevy 的一系列文章。

<!-- more -->

# app

一个 app 包含一个主世界和多个子世界。
当主世界的调度执行完之后，子世界的的 extra 函数和 update 才会执行。
