# `Peer`: 连接上游

在`upstream_peer()`阶段，用户最终应返回一个`Peer`对象，此对象定义了如何连接到相应的上游。

## `Peer`
`HttpPeer`定义了连接到哪个上游。
| 属性      | 说明        |
| ------------- |-------------|
|address: `SocketAddr`| 上游的 IP:Port |
|scheme: `Scheme`| Http 或 Https |
|sni: `String`| SNI, 仅对Https |
|proxy: `Option<Proxy>`| 通过[CONNECT代理](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/CONNECT)对请求进行代理时的设置 |
|client_cert_key: `Option<Arc<CertKey>>`| 通过mTLS连接到上游时所用的客户端证书。双向认证 |
|options: `PeerOptions`| 说明如下 |


## `PeerOptions`
`PeerOptions`定义了如何连接到上游
| 属性      | 说明        |
| ------------- |-------------|
|bind_to: `Option<InetSocketAddr>`| 作为客户端向上游发起请求时应当绑定的本地源IP地址 |
|connection_timeout: `Option<Duration>`| 申请建立连接的超时时间，到期后仍未建立成功则放弃 |
|total_connection_timeout: `Option<Duration>`| 申请建立连接的总超时时间，包括TLS握手时间 |
|read_timeout: `Option<Duration>`| **单次**执行`read()`的超时时间。每次执行`read()`后计时器都会被重置 |
|idle_timeout: `Option<Duration>`| 连接空闲超时时间，连续空闲一段时间后连接会被回收 |
|write_timeout: `Option<Duration>`| 通过`write()`向上游写入内容的超时时间 |
|verify_cert: `bool`| 是否需要校验上游的服务器证书的有效性 |
|verify_hostname: `bool`| 是否需要校验上游的服务器证书的CN与SNI能否匹配 |
|alternative_cn: `Option<String>`| 校验上游的服务器证书的CN，如果匹配则接受证书 |
|alpn: `ALPN`| ALPN(应用层协议协商)首选项, http1.1 和/或 http2 |
|ca: `Option<Arc<Box<[X509]>>>`| 校验上游的服务器证书时所用的根CA |
|tcp_keepalive: `Option<TcpKeepalive>`| TCP连接保活设置 |

## Examples
TBD
