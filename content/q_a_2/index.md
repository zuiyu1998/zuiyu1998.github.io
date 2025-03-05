+++
title = "切片和原数组的长度关系"
date = 2025-03-05

[taxonomies]
tags = ["q_a", "rust", "mutex"]
+++

场景：

一个数组的切片长度和原数组的长度关系是什么样的？

- 两者一样
- 各自维护自己的长度信息

<!-- more -->

# 验证

设计一个简单的代码如下:

```rust
fn main() {
    let bytes = vec![1, 2, 3];

    println!("bytes len:{}", bytes.len());

    let slice = &bytes[0..2];
    println!("slice len:{}", slice.len());
}

```

运行程序后，可以看到终端输出 3 和 2.可以看到 slice 维护了自己的长度信息。
