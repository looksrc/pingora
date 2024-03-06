# How to return errors

为了方便处理错误，`pingora-error`库导出了一个自定义的`Result`类型，其它Pingora库都使用这个类型。

`Result`中使用的`Error`结构体可以包装任意错误。让用户可以对底层错误源进行打标并附加自定义的上下文信息。

用户通常需要通过传播一个已有的错误或创建一个全新的错误用于最终的返回。`pingora-error`内置的函数让这个工作变得更方便。

## 例子

例如，当请求中缺失了某个预期的头字段时，可以返回错误：

```rust
fn validate_req_header(req: &RequestHeader) -> Result<()> {
    // 验证HTTP头部是否含有`host`字段
    req.headers()
        .get(http::header::HOST)
        .ok_or_else(|| Error::explain(InvalidHTTPHeader, "No host header detected"))
}

impl MyServer {
    pub async fn handle_request_filter(
        &self,
        http_session: &mut Session,
        ctx: &mut CTX,
    ) -> Result<bool> {
        validate_req_header(session.req_header()?).or_err(HTTPStatus(400), "Missing required headers")?;
        Ok(true)
    }
}
```

当HTTP请求头中缺失了`host`字段，`validate_req_header`会返回一个`Error`，使用`Error::explain`函数联合`InvalidHTTPHeader`类型，创建了一个错误并附加了一些可以记录在错误日志中的上下文信息。

此错误最终会传播到`request_filter`中，然后通过`or_err`处理后返回`HTTPStatus`。
(最终在`fail_to_proxy()`阶段，会将此错误记录到日志，并给下游客户端发送`400 Bad Request`)。

同时，最原始的错误信息会被记录到错误日志中。`or_err`将原始的错误包装为一个新错误，并附加了一些额外的信息，但是`Error`的`Display`实现仍然会打印错误链。

## 指导原则

一个错误包含：
- 一个 _type_ （如`ConnectionClosed`)
- 一个 _source_ （如`Upstream`, `Downstream`, `Internal`)
- 一个可选的 _cause_ (一个被包装的错误)
- 一个 _context_ (用户提供的任意字符串)

一个最小的错误可以通过函数创建，如`new_in` / `new_up` / `new_down`，每个函数指定了一个需要用户提供类型的源错误。

通常来讲:
* 要创建一个不需要直接原因只需要提供上下文的新错误，则使用`Error::explain`。也可以在`Result`上调用`explain_err`将内部的错误替换为新的错误。
* To wrap a causing error in a new one with more context, use `Error::because`. You can also use `or_err` on a `Result` to replace the potential error inside it by wrapping the original one.
要将一个源错误包装到一个新错误并附加更多上下文，使用`Error::because`。也可以在`Result`上调用`or_err`

## 重试

错误是可重试的。如果一个错误是可重试的，Pingora代理允许重试上游请求。一些错误只允许在[复用](pooling.md)的连接上重试，比如在处理远端已经丢弃了服务器尝试复用的连接时。

默认情况下，新建的`Error`可以持有它的直接源错误的重试状态，或者在未指定重试的情况下被认为是不可重试的。
