# 失败处理和故障转移

Pingora 代理可以在整个代理请求处理的过程中，让用户定义如何处理失败。

如果在响应头发送给下游之前发生了错误，用户可以有以下几个选择：
1. 发送错误页面给下游，然后放弃此请求
2. 重试同一个上游
3. 重试另一个上游，如果合适的话。

另外，一旦响应头已经发送给下游，代理就只能记录一个错误日志并放弃此请求，没法做其他事情了。


## 失败重试 / 故障转移
为了实现重试或故障转移，需要在`fail_to_connect()` / `error_while_proxy()`阶段中将错误标记为“可重试”。
对于故障转移，`fail_to_connect() / error_while_proxy()`还需要更新`CTX`，告诉`upstream_peer()`不要再使用同一个`Peer`。

### 安全性
正常讲，幂等的HTTP请求，比如`GET`，可以安全的重试。其它请求，例如`POST`，在请求已经发出的情况下，重试时不能保证安全。
如进入了`fail_to_connect()`阶段，Pingora代理可以确保没有向上游发送过任何内容，此时重试是安全的。
不建议用户在`error_while_proxy()`阶段以后重试非幂等的HTTP请求，除非用户对上游足够了解，知道此重试操作是安全的。

### 例子
在下面的例子中，我们在`CTX`中设置了一个`tries`变量跟踪已经重连的次数。
在`upstream_peer`中计算`Peer`时，校验重连次数`tries`如果小于1次则连接到192.0.2.1。
连接失败时，在`fail_to_connect`中将`tries`增一，并设置`e.set_retry(true)`将此错误标记为可重试。
重试时，再次进入了`upstream_peer`，并连接到1.1.1.1。
如果连接1.1.1.1也失败了，则直接返回502，因为在`fail_to_connect`中只有当`tries`为零时才会设置重试`e.set_retry(true)`。

```Rust
pub struct MyProxy();

pub struct MyCtx {
    tries: usize,
}

#[async_trait]
impl ProxyHttp for MyProxy {
    type CTX = MyCtx;
    fn new_ctx(&self) -> Self::CTX {
        MyCtx { tries: 0 }
    }

    fn fail_to_connect(
        &self,
        _session: &mut Session,
        _peer: &HttpPeer,
        ctx: &mut Self::CTX,
        mut e: Box<Error>,
    ) -> Box<Error> {
        if ctx.tries > 0 {
            return e;
        }
        ctx.tries += 1;
        e.set_retry(true);
        e
    }

    async fn upstream_peer(
        &self,
        _session: &mut Session,
        ctx: &mut Self::CTX,
    ) -> Result<Box<HttpPeer>> {
        let addr = if ctx.tries < 1 {
            ("192.0.2.1", 443)
        } else {
            ("1.1.1.1", 443)
        };

        let mut peer = Box::new(HttpPeer::new(addr, true, "one.one.one.one".to_string()));
        peer.options.connection_timeout = Some(Duration::from_millis(100));
        Ok(peer)
    }
}
```
