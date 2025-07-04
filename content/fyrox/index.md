+++
title = "论Fyrox中Node的实现"
date = 2025-06-13

[taxonomies]
tags = ["Fyrox", "Node"]
+++

论 Fyrox 中 Node 的实现

<!-- more -->

# Node

Node 有多个种类，同时每个不同种类 Node 拥有共同的数据和操作。

当前方案: 通过 deref 指向 Node 共有的结构，继承共有的方法和数据。

```rust
#[derive(Debug)]
pub struct Node(pub(crate) Box<dyn NodeTrait>);

pub trait BaseNodeTrait: NodeAsAny + Debug + Deref<Target = Base> + DerefMut + Send {
    fn clone_box(&self) -> Node;
}

pub trait NodeTrait: BaseNodeTrait + Reflect + Visit + ComponentProvider {}

```
