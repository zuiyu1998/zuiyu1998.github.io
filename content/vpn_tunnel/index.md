+++
title = "vpn tunnel"
date = 2024-08-19

[taxonomies]
tags = ["vpn", "easytier"]
+++

vpn 隧道的深入分析

<!-- more -->

# 目标

vpn 使用隧道进行加密通信。因此它是双工信道。

- 隧道是多个类型的
- 每个隧道的流转的 ZCPacket 是不一致的
- 每个 ZCPacket 需要支持互相转化

# packet_def

vpn 中流转的基础数据是 ZCPacket。

```rust
#[derive(Debug, Clone)]
pub struct ZCPacket {
    inner: BytesMut,
    packet_type: ZCPacketType,
}
```

ZCPacket 的实际数据为 BytesMut。BytesMut 中的数据分为如下:

- wg_tunnel_header wireguard
- udp_tunnel_header udp
- tcp_tunnel_header tcp
- dummy_tunnel_header
- peer_manager_header peer_manager

# PeerManagerHeader

PeerManagerHeader 描述了每个连接之间的必要信息。

```rust
#[repr(C, packed)]
#[derive(IntoBytes, FromBytes, Clone, Debug, Default, KnownLayout, Immutable)]
pub struct PeerManagerHeader {
    pub from_peer_id: U32<DefaultEndian>,
    pub to_peer_id: U32<DefaultEndian>,
    pub packet_type: u8,
    pub flags: u8,
    pub forward_counter: u8,
    reserved: u8,
    pub len: U32<DefaultEndian>,
}

#[derive(IntoBytes, FromZeros, Clone, Debug)]
#[repr(u8)]
pub enum PacketType {
    Invalid = 0,
    Data = 1,
    HandShake = 2,
    RoutePacket = 3,
    Ping = 4,
    Pong = 5,
    TaRpc = 6,
    Route = 7,
    RpcReq = 8,
    RpcResp = 9,
    ForeignNetworkPacket = 10,
}

bitflags::bitflags! {
    struct PeerManagerHeaderFlags: u8 {
        const ENCRYPTED = 0b0000_0001;
        const LATENCY_FIRST = 0b0000_0010;
        const EXIT_NODE = 0b0000_0100;
        const NO_PROXY = 0b0000_1000;

        const _ = !0;
    }
}
```

# TCPTunnelHeader

tcp 隧道中 packet 的 header 数据。

```rust
#[repr(C, packed)]
#[derive(Clone, Debug, Default, Immutable, FromBytes, KnownLayout, IntoBytes)]
pub struct TCPTunnelHeader {
    pub len: U32<DefaultEndian>,
}
```

# 定义隧道

一个隧道的定义如下:

```rust
///隧道
pub trait Tunnel: Send {
    fn split(&self) -> (Pin<Box<dyn ZCPacketStream>>, Pin<Box<dyn ZCPacketSink>>);
    fn info(&self) -> Option<TunnelInfo>;
}

pub trait ZCPacketStream: Stream<Item = StreamItem> + Send {}
pub trait ZCPacketSink: Sink<SinkItem, Error = Error> + Send {}

pub type StreamItem = Result<ZCPacket, Error>;
pub type SinkItem = ZCPacket;

#[derive(Debug, Error)]
pub enum Error {
    #[error("invalid packet. msg: {0}")]
    InvalidPacket(String),
    #[error("io error")]
    IOError(#[from] std::io::Error),
    #[error("invalid protocol: {0}")]
    InvalidProtocol(String),
    #[error("no dns record found")]
    NoDnsRecordFound(IpVersion),
}
```

这里主要的设计就是 split 函数。split 函数会返回两个信道。

# TunnelListener

隧道服务器，用于绑定和生成可用的隧道，类似于 Tcp 的架构。

- 绑定
- 隧道生成

```rust
pub trait TunnelListener: Send {
    async fn listen(&mut self) -> Result<(), Error>;
    async fn accept(&mut self) -> Result<Box<dyn Tunnel>, Error>;
    fn local_url(&self) -> Url;
    fn get_conn_counter(&self) -> Arc<Box<dyn TunnelConnCounter>> {
        #[derive(Debug)]
        struct FakeTunnelConnCounter {}
        impl TunnelConnCounter for FakeTunnelConnCounter {
            fn get(&self) -> Option<u32> {
                None
            }
        }
        Arc::new(Box::new(FakeTunnelConnCounter {}))
    }
}
```

# TunnelConnector

隧道客户端，用于连接服务器，生成唯一的隧道。

- 绑定

```rust
//客户端
#[async_trait]
pub trait TunnelConnector: Send {
    async fn connect(&mut self) -> Result<Box<dyn Tunnel>, Error>;
    fn remote_url(&self) -> Url;
    fn set_bind_addrs(&mut self, _addrs: Vec<SocketAddr>) {}
    fn set_ip_version(&mut self, _ip_version: IpVersion) {}
}
```

# tcp 隧道

tcp 分包和写入
定义一个对象实现 Stream trait 来解析 tcp 的数据。

```rust
// a length delimited codec for async reader
pin_project! {
    pub struct FramedReader<R> {
        #[pin]
        reader: R,
        buf: BytesMut,
        max_packet_size: usize,
        associate_data: Option<Box<dyn Any + Send + 'static>>,
    }
}
```

reader 是 tokio 用来读取 Buf 的泛型,buf 当前的数据，max_packet_size，最长长度配置。
这里要注意的是分包逻辑。
