+++
title = "论bevy window的设计"
date = 2024-07-31

[taxonomies]
tags = ["bevy", "rust", "window", "winit"]
+++

bevy 中 window 的设计。

<!-- more -->

# 目标

bevy 使用 winit 作为窗口库，可以高效的使用 winit 中 Window 对象作为窗口的对象，但实际上，bevy 却单独实现了一个 bevy_window 库来抽象，这里的目的并不复杂，那就是保留一种可能性，那就是更改窗口库的可能。也就是窗口库可以使用 winit，也可以用其他。窗口相对于窗口库是一对多的关系。

# 实现

这里比较特殊的实现是，window 的抽象并不是添加更多信息，而是直接获取了操作系统的窗口 handle。这里使用了 raw-window-handle 库。

window 的 wrapper 抽象如下：

```rust
#[derive(Debug)]
pub struct WindowWrapper<W> {
    reference: Arc<dyn Any + Send + Sync>,
    ty: PhantomData<W>,
}

impl<W: Send + Sync + 'static> WindowWrapper<W> {
    /// Creates a `WindowWrapper` from a window.
    pub fn new(window: W) -> WindowWrapper<W> {
        WindowWrapper {
            reference: Arc::new(window),
            ty: PhantomData,
        }
    }
}

impl<W: 'static> Deref for WindowWrapper<W> {
    type Target = W;

    fn deref(&self) -> &Self::Target {
        self.reference.downcast_ref::<W>().unwrap()
    }
}

```

这里的泛型 W 可以是任何窗口库的 Window 对象。但是实际使用的其实下面这个抽象。

```rust
#[derive(Debug, Clone, Component)]
pub struct RawHandleWrapper {
    _window: Arc<dyn Any + Send + Sync>,
    /// Raw handle to a window.
    pub window_handle: RawWindowHandle,
    /// Raw handle to the display server.
    pub display_handle: RawDisplayHandle,
}

impl RawHandleWrapper {
    /// Creates a `RawHandleWrapper` from a `WindowWrapper`.
    pub fn new<W: HasWindowHandle + HasDisplayHandle + 'static>(
        window: &WindowWrapper<W>,
    ) -> Result<RawHandleWrapper, HandleError> {
        Ok(RawHandleWrapper {
            _window: window.reference.clone(),
            window_handle: window.window_handle()?.as_raw(),
            display_handle: window.display_handle()?.as_raw(),
        })
    }

    /// Returns a [`HasWindowHandle`] + [`HasDisplayHandle`] impl, which exposes [`WindowHandle`] and [`DisplayHandle`].
    ///
    /// # Safety
    ///
    /// Some platforms have constraints on where/how this handle can be used. For example, some platforms don't support doing window
    /// operations off of the main thread. The caller must ensure the [`RawHandleWrapper`] is only used in valid contexts.
    pub unsafe fn get_handle(&self) -> ThreadLockedRawWindowHandleWrapper {
        ThreadLockedRawWindowHandleWrapper(self.clone())
    }
}
```

通过 new 获取到的对象构造一个安全 RawHandleWrapper，获取操作系统的窗口 handle。

当然还有一个多线程使用的版本。定义如下:

```rust
#[derive(Debug, Clone, Component)]
pub struct RawHandleWrapperHolder(pub Arc<Mutex<Option<RawHandleWrapper>>>);

```

# 体悟

WindowWrapper 实例化了一个任何一个窗口库的抽象。RawHandleWrapper 则是对其 WindowWrapper 的类型进行了擦除，只保留了最基础的窗口 handle。哪怕使用其他窗口库，也可以复用 window 中的逻辑。
