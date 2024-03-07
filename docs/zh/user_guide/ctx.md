# 在各阶段共享状态：`CTX`

## 使用 `CTX`
用户在请求的不同阶段实现的自定义过滤器之间不能直接交互。为了能在过滤器之间共享信息和状态，可以定义一个`CTX`结构体。
每个请求都拥有一个自己的`CTX`对象。所有过滤器都能读取和更新`CTX`对象的内容。在请求结束时会遗弃CTX对象。

### 例子

在下面的例子中，代理在`request_filter`阶段解析请求头部的`beta-flag`字段并将信息存入`MyCtx`中，在接下来的`upstream_peer`阶段通过`MyCtx`中的内容判断请求应该转发到哪个后端。(技术上讲，也可以在`upstream_peer`阶段解析头部，这里提前解析只是为了演示`CTX`的用法)。

```Rust
pub struct MyProxy();

pub struct MyCtx {
    beta_user: bool,
}

fn check_beta_user(req: &pingora_http::RequestHeader) -> bool {
    // 用一些简单的逻辑校验用户是否为beta用户
    req.headers.get("beta-flag").is_some()
}

#[async_trait]
impl ProxyHttp for MyProxy {
    type CTX = MyCtx;
    fn new_ctx(&self) -> Self::CTX {
        MyCtx { beta_user: false }
    }

    async fn request_filter(&self, session: &mut Session, ctx: &mut Self::CTX) -> Result<bool> {
        ctx.beta_user = check_beta_user(session.req_header());
        Ok(false)
    }

    async fn upstream_peer(
        &self,
        _session: &mut Session,
        ctx: &mut Self::CTX,
    ) -> Result<Box<HttpPeer>> {
        let addr = if ctx.beta_user {
            info!("I'm a beta user");
            ("1.0.0.1", 443)
        } else {
            ("1.1.1.1", 443)
        };

        let peer = Box::new(HttpPeer::new(addr, true, "one.one.one.one".to_string()));
        Ok(peer)
    }
}
```

## 请求间共享状态
在不同请求间共享诸如计数、缓存、额外信息之类的状态比较常见。
在Pingora中，不同请求间共享资源和数据时没有什么特殊的方式，可以使用`Arc`、`static`或其他一些可用的机制。

### 例子
让我们来继续演进上面的例子，添加跟踪beta用户的访问次数和所有用户的访问次数的功能。计数变量可以定义在`MyProxy`中或者全局变量中。由于计数变量需要并发访问，因此需要加上互斥锁。

```Rust
// 整个代理的全局请求计数
static REQ_COUNTER: Mutex<usize> = Mutex::new(0);

pub struct MyProxy {
    // 当前服务接受到的beta用户的请求计数
    beta_counter: Mutex<usize>, // 也可以用AtomicUsize
}

pub struct MyCtx {
    beta_user: bool,
}

fn check_beta_user(req: &pingora_http::RequestHeader) -> bool {
    // 用一些简单的逻辑校验用户是否为beta用户
    req.headers.get("beta-flag").is_some()
}

#[async_trait]
impl ProxyHttp for MyProxy {
    type CTX = MyCtx;
    fn new_ctx(&self) -> Self::CTX {
        MyCtx { beta_user: false }
    }

    async fn request_filter(&self, session: &mut Session, ctx: &mut Self::CTX) -> Result<bool> {
        ctx.beta_user = check_beta_user(session.req_header());
        Ok(false)
    }

    async fn upstream_peer(
        &self,
        _session: &mut Session,
        ctx: &mut Self::CTX,
    ) -> Result<Box<HttpPeer>> {
        let mut req_counter = REQ_COUNTER.lock().unwrap();
        *req_counter += 1;

        let addr = if ctx.beta_user {
            let mut beta_count = self.beta_counter.lock().unwrap();
            *beta_count += 1;
            info!("I'm a beta user #{beta_count}");
            ("1.0.0.1", 443)
        } else {
            info!("I'm an user #{req_counter}");
            ("1.1.1.1", 443)
        };

        let peer = Box::new(HttpPeer::new(addr, true, "one.one.one.one".to_string()));
        Ok(peer)
    }
}
```

完整的例子在[`pingora-proxy/examples/ctx.rs`](../../../pingora-proxy/examples/ctx.rs)，通过下面的命令启动代理:
```
RUST_LOG=INFO cargo run --example ctx
```