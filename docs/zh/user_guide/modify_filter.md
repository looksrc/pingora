# 例子: 控制请求

在这一章，将介绍如何路由、修改、拒绝请求。

## 路由
请求携带的任何信息都可以用来做路由决策。Pingora 对用户如何实现自己的路由逻辑不强加任何限制。

在下面的例子中，如果请求路径以`/family/`开头，则代理将流量转发给1.0.0.1，其它的流量转发给1.1.1.1。

```Rust
pub struct MyGateway;

#[async_trait]
impl ProxyHttp for MyGateway {
    type CTX = ();
    fn new_ctx(&self) -> Self::CTX {}

    async fn upstream_peer(
        &self,
        session: &mut Session,
        _ctx: &mut Self::CTX,
    ) -> Result<Box<HttpPeer>> {
        let addr = if session.req_header().uri.path().starts_with("/family/") {
            ("1.0.0.1", 443)
        } else {
            ("1.1.1.1", 443)
        };

        info!("connecting to {addr:?}");

        let peer = Box::new(HttpPeer::new(addr, true, "one.one.one.one".to_string()));
        Ok(peer)
    }
}
```


## 修改头部

请求头和响应头都可以在相应的阶段进行添加、修改、删除。
下面的例子中，我们将在`response_filter`阶段添加逻辑，更新`Server`字段，删除`alt-svc`字段。

```Rust
#[async_trait]
impl ProxyHttp for MyGateway {
    ...
    async fn response_filter(
        &self,
        _session: &mut Session,
        upstream_response: &mut ResponseHeader,
        _ctx: &mut Self::CTX,
    ) -> Result<()>
    where
        Self::CTX: Send + Sync,
    {
        // 更新Server字段
        upstream_response
            .insert_header("Server", "MyGateway")
            .unwrap();
        // 因为不支持H3所以删除alt-svc字段
        upstream_response.remove_header("alt-svc");

        Ok(())
    }
}
```

## 返回错误页面

在某些情况下不能对流量进行代理，比如认证失败，此时可以让代理直接返回错误页面。

```Rust
fn check_login(req: &pingora_http::RequestHeader) -> bool {
    // 在此处实现你的登录校验逻辑
    req.headers.get("Authorization").map(|v| v.as_bytes()) == Some(b"password")
}

#[async_trait]
impl ProxyHttp for MyGateway {
    ...
    async fn request_filter(&self, session: &mut Session, _ctx: &mut Self::CTX) -> Result<bool> {
        if session.req_header().uri.path().starts_with("/login")
            && !check_login(session.req_header())
        {
            let _ = session.respond_error(403).await;
            // true: 告诉代理HTTP响应内容已经写入完成
            return Ok(true);
        }
        Ok(false)
    }
```
## 记录日志

可以在Pingora的`logging`阶段添加记录日志的逻辑。每个请求在被Pingora代理处理完成前，无论请求成功还是失败，都会执行到`logging`阶段。

在下例中，我们为代理添加了Prometheus指标和访问日志。同时还在其他端口上启动了一个Prometheus指标服务器，用于指标抓取。


``` Rust
pub struct MyGateway {
    req_metric: prometheus::IntCounter,
}

#[async_trait]
impl ProxyHttp for MyGateway {
    ...
    async fn logging(
        &self,
        session: &mut Session,
        _e: Option<&pingora::Error>,
        ctx: &mut Self::CTX,
    ) {
        let response_code = session
            .response_written()
            .map_or(0, |resp| resp.status.as_u16());
        // 输出访问日志
        info!(
            "{} response code: {response_code}",
            self.request_summary(session, ctx)
        );

        // 更新Prometheus指标
        self.req_metric.inc();
    }

fn main() {
    // 此处省略了启动MyGateway的代码
    ...
    // 启动Prometheus指标服务器
    let mut prometheus_service_http =
        pingora::services::listening::Service::prometheus_http_service();
    prometheus_service_http.add_tcp("127.0.0.1:6192");
    my_server.add_service(prometheus_service_http);

    my_server.run_forever();
}
```