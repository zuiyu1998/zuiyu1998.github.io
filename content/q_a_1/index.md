+++
title = "在多线程中有高计算场景中如何持有锁"
date = 2025-02-24

[taxonomies]
tags = ["q_a", "rust", "mutex"]
+++

场景：

- 多线程使用
- 有一个或多个线程有高计算使用

在这种情况下如何使用锁？

<!-- more -->

# 方案

通常情况下分两种情况:

1. 在高计算场景下多次获取锁
2. 在高计算场景下整体获取锁

情况 1 符合一般的锁的使用场景，缩小临界区。避免死锁。那情况 2 可使用吗？

如果是整体使用锁，且这个锁在线程内只使用一次没问题。需要注意的是在单线程内对同一个锁进行多次申请。
测试代码如下:

```rust
use std::{
    sync::{Arc, Mutex},
    thread::{self, sleep},
    time::Duration,
};

#[derive(Debug, Default, Clone)]
pub struct State {
    count: Arc<Mutex<usize>>,
}

impl State {
    pub fn computed(&self) {
        println!("computed");
        let mut count = self.count.lock().unwrap();

        sleep(Duration::from_secs(2));

        *count += 1;
    }

    pub fn print(&self) {
        let mut count = self.count.lock().unwrap();
        *count += 1;

        println!("{}", count);
    }
}

fn main() {
    let state = State::default();

    let state_clone = state.clone();
    thread::spawn(move || {
        state_clone.computed();
    });


    let state_clone = state.clone();
    thread::spawn(move || {
        state_clone.computed();
    });

    for _ in 0..1000 {
        state.print();
    }

    sleep(Duration::from_secs(3));
}

```
