+++
title = "论bevy asset的设计"
date = 2024-07-28

[taxonomies]
tags = ["bevy", "rust", "asset"]
+++

bevy 中 asset 系统的设计。

<!-- more -->

# 目标

资源加载器负责资源的加载和卸载。

资源是什么？以及如何从文件系统中加载资源？资源和文件的对应关系是什么？

bevy 中的资源由 asset trait 定义。它的定义如下:

```rust
pub trait Asset: VisitAssetDependencies + TypePath + Send + Sync + 'static {}

```

在 bevy 中资源加载器的定义为 AssetServer。不需要关注 AssetServer 的具体实现。而是从具体的 load 函数出发，它的定义如下:

```rust
pub(crate) fn load_with_meta_transform<'a, A: Asset, G: Send + Sync + 'static>(
    &self,
    path: impl Into<AssetPath<'a>>,
    meta_transform: Option<MetaTransform>,
    guard: G,
) -> Handle<A> {
    let path = path.into().into_owned();
    let (handle, should_load) = self.data.infos.write().get_or_create_path_handle::<A>(
        path.clone(),
        HandleLoadingMode::Request,
        meta_transform,
    );

    if should_load {
        let owned_handle = Some(handle.clone().untyped());
        let server = self.clone();
        IoTaskPool::get()
            .spawn(async move {
                if let Err(err) = server.load_internal(owned_handle, path, false, None).await {
                    error!("{}", err);
                }
                drop(guard);
            })
            .detach();
    }

    handle
}
```

get_or_create_path_handle 创建一个索引，可以用这个索引在 Assets 中获取具体的资源数据。其中 load_internal 才是真正从文件系统中加载数据的逻辑，load_internal 的定义如下:

```rust
    async fn load_internal<'a>(
        &self,
        input_handle: Option<UntypedHandle>,
        path: AssetPath<'a>,
        force: bool,
        meta_transform: Option<MetaTransform>,
    ) -> Result<UntypedHandle, AssetLoadError> {
        let asset_type_id = input_handle.as_ref().map(UntypedHandle::type_id);

        let path = path.into_owned();
        let path_clone = path.clone();
        let (mut meta, loader, mut reader) = self
            .get_meta_loader_and_reader(&path_clone, asset_type_id)
            .await
            .map_err(|e| {
                // if there was an input handle, a "load" operation has already started, so we must produce a "failure" event, if
                // we cannot find the meta and loader
                if let Some(handle) = &input_handle {
                    self.send_asset_event(InternalAssetEvent::Failed {
                        id: handle.id(),
                        path: path.clone_owned(),
                        error: e.clone(),
                    });
                }
                e
            })?;

        // This contains Some(UntypedHandle), if it was retrievable
        // If it is None, that is because it was _not_ retrievable, due to
        //    1. The handle was not already passed in for this path, meaning we can't just use that
        //    2. The asset has not been loaded yet, meaning there is no existing Handle for it
        //    3. The path has a label, meaning the AssetLoader's root asset type is not the path's asset type
        //
        // In the None case, the only course of action is to wait for the asset to load so we can allocate the
        // handle for that type.
        //
        // TODO: Note that in the None case, multiple asset loads for the same path can happen at the same time
        // (rather than "early out-ing" in the "normal" case)
        // This would be resolved by a universal asset id, as we would not need to resolve the asset type
        // to generate the ID. See this issue: https://github.com/bevyengine/bevy/issues/10549
        let handle_result = match input_handle {
            Some(handle) => {
                // if a handle was passed in, the "should load" check was already done
                Some((handle, true))
            }
            None => {
                let mut infos = self.data.infos.write();
                let result = infos.get_or_create_path_handle_internal(
                    path.clone(),
                    path.label().is_none().then(|| loader.asset_type_id()),
                    HandleLoadingMode::Request,
                    meta_transform,
                );
                unwrap_with_context(result, loader.asset_type_name())
            }
        };

        let handle = if let Some((handle, should_load)) = handle_result {
            if path.label().is_none() && handle.type_id() != loader.asset_type_id() {
                error!(
                    "Expected {:?}, got {:?}",
                    handle.type_id(),
                    loader.asset_type_id()
                );
                return Err(AssetLoadError::RequestedHandleTypeMismatch {
                    path: path.into_owned(),
                    requested: handle.type_id(),
                    actual_asset_name: loader.asset_type_name(),
                    loader_name: loader.type_name(),
                });
            }
            if !should_load && !force {
                return Ok(handle);
            }
            Some(handle)
        } else {
            None
        };
        // if the handle result is None, we definitely need to load the asset

        let (base_handle, base_path) = if path.label().is_some() {
            let mut infos = self.data.infos.write();
            let base_path = path.without_label().into_owned();
            let (base_handle, _) = infos.get_or_create_path_handle_untyped(
                base_path.clone(),
                loader.asset_type_id(),
                loader.asset_type_name(),
                HandleLoadingMode::Force,
                None,
            );
            (base_handle, base_path)
        } else {
            (handle.clone().unwrap(), path.clone())
        };

        if let Some(meta_transform) = base_handle.meta_transform() {
            (*meta_transform)(&mut *meta);
        }

        match self
            .load_with_meta_loader_and_reader(&base_path, meta, &*loader, &mut *reader, true, false)
            .await
        {
            Ok(loaded_asset) => {
                let final_handle = if let Some(label) = path.label_cow() {
                    match loaded_asset.labeled_assets.get(&label) {
                        Some(labeled_asset) => labeled_asset.handle.clone(),
                        None => {
                            let mut all_labels: Vec<String> = loaded_asset
                                .labeled_assets
                                .keys()
                                .map(|s| (**s).to_owned())
                                .collect();
                            all_labels.sort_unstable();
                            return Err(AssetLoadError::MissingLabel {
                                base_path,
                                label: label.to_string(),
                                all_labels,
                            });
                        }
                    }
                } else {
                    // if the path does not have a label, the handle must exist at this point
                    handle.unwrap()
                };

                self.send_loaded_asset(base_handle.id(), loaded_asset);
                Ok(final_handle)
            }
            Err(err) => {
                self.send_asset_event(InternalAssetEvent::Failed {
                    id: base_handle.id(),
                    error: err.clone(),
                    path: path.into_owned(),
                });
                Err(err)
            }
        }
    }

```

get_meta_loader_and_reader 获取加载器。加载器实际负责从文件系统加载数据。bevy 中加载器通过两个 trait 来完成设计。这两个 trait 如下:

```rust
pub trait ErasedAssetLoader: Send + Sync + 'static {
    /// Asynchronously loads the asset(s) from the bytes provided by [`Reader`].
    fn load<'a>(
        &'a self,
        reader: &'a mut dyn Reader,
        meta: Box<dyn AssetMetaDyn>,
        load_context: LoadContext<'a>,
    ) -> BoxedFuture<
        'a,
        Result<ErasedLoadedAsset, Box<dyn std::error::Error + Send + Sync + 'static>>,
    >;

    /// Returns a list of extensions supported by this asset loader, without the preceding dot.
    fn extensions(&self) -> &[&str];
    /// Deserializes metadata from the input `meta` bytes into the appropriate type (erased as [`Box<dyn AssetMetaDyn>`]).
    fn deserialize_meta(&self, meta: &[u8]) -> Result<Box<dyn AssetMetaDyn>, DeserializeMetaError>;
    /// Returns the default meta value for the [`AssetLoader`] (erased as [`Box<dyn AssetMetaDyn>`]).
    fn default_meta(&self) -> Box<dyn AssetMetaDyn>;
    /// Returns the type name of the [`AssetLoader`].
    fn type_name(&self) -> &'static str;
    /// Returns the [`TypeId`] of the [`AssetLoader`].
    fn type_id(&self) -> TypeId;
    /// Returns the type name of the top-level [`Asset`] loaded by the [`AssetLoader`].
    fn asset_type_name(&self) -> &'static str;
    /// Returns the [`TypeId`] of the top-level [`Asset`] loaded by the [`AssetLoader`].
    fn asset_type_id(&self) -> TypeId;
}

pub trait AssetLoader: Send + Sync + 'static {
    /// The top level [`Asset`] loaded by this [`AssetLoader`].
    type Asset: crate::Asset;
    /// The settings type used by this [`AssetLoader`].
    type Settings: Settings + Default + Serialize + for<'a> Deserialize<'a>;
    /// The type of [error](`std::error::Error`) which could be encountered by this loader.
    type Error: Into<Box<dyn std::error::Error + Send + Sync + 'static>>;
    /// Asynchronously loads [`AssetLoader::Asset`] (and any other labeled assets) from the bytes provided by [`Reader`].
    fn load<'a>(
        &'a self,
        reader: &'a mut dyn Reader,
        settings: &'a Self::Settings,
        load_context: &'a mut LoadContext,
    ) -> impl ConditionalSendFuture<Output = Result<Self::Asset, Self::Error>>;

    /// Returns a list of extensions supported by this [`AssetLoader`], without the preceding dot.
    /// Note that users of this [`AssetLoader`] may choose to load files with a non-matching extension.
    fn extensions(&self) -> &[&str] {
        &[]
    }
}

```

AssetLoader 负责加载器的具体实现，从文件系统中读取内容，并转化成 asset 对象，ErasedAssetLoader，负责保存加载器的指针，同时擦除 Asset 的类型信息。ErasedLoadedAsset 保存了资源信息。
load_internal 函数中 load_with_meta_loader_and_reader 负责实际加载数据，loaded_asset 就是擦除类型之后的资源。send_loaded_asset 方法将加载后资源交给其他系统进行处理，

handle_internal_asset_events 用于加载后的数据处理。
