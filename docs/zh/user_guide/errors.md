# How to return errors

为了方便处理错误，`pingora-error`库导出了一个自定义的`Result`类型，其它Pingora库都在使用这个类型。

`Result`的`Err`变体中使用的`Error`结构体可以包装任意错误，使用户可以记录底层错误并附加新的上下文信息。

用户通常需要通过传播一个已有的错误或创建一个全新的错误用于最终的返回。`pingora-error`内置的函数可以辅助这些操作。

## 例子

例如，当请求中缺失了某个想要的头字段时，可以返回错误：

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

在例子中，当HTTP请求头中缺失了`host`字段，`validate_req_header`会返回一个`Result::Err(e)`，
内部的错误实例是使用`Error::explain`函数传入`InvalidHTTPHeader`错误类型和上下文信息所创建的。

`Result::Err(e)`最终会传播到`request_filter`中，然后调用`or_err`方法会创建一个`etype`为`HTTPStatus(400)`、`cause`为e的新的错误实例。
(最终在`fail_to_proxy()`阶段，会将此错误记录到日志，并给下游客户端发送`400 Bad Request`)。

最原始的错误信息也会被记录到错误日志中。`or_err`以原始错误作为`cause`并提供了`context`信息创建新错误，但是`Error`的`Display`实现仍然会打印错误链。

## 指导原则

一个错误包含：
- 错误类型 _etype_ （如`ConnectionClosed`)
- 错误来源 _esource_ （如`Upstream`, `Downstream`, `Internal`，`Unset`)
- 错误根因 _cause_ 可选的，(是一个被包装的错误)
- 重试类型 _retry_ （如`Decided(bool)`，`ReusedOnly`)
- 错误上下文 _context_ (用户提供的任意字符串)

一个最小的错误可以通过函数创建，如`new_in` / `new_up` / `new_down`，每个函数需要指定一个需要用户提供类型的源错误。

通常来讲:
* 要想创建一个不带根因( _source_ 为Unset)只提供上下文的错误(`Box<Error>`)，调用`Error::explain`。<br>
在`pingora_error::Result`上调用`explain_err(Self,ErrorType,F)`，会将错误替换为通过`explain`函数生成的新错误。新错误的上下文信息是将原错误传入闭包`F`后计算得出的。
* 想要将源错误作为 _source_ 包装进一个新错误(`Box<Error>`)，并为新错误附加上下文信息时,使用`Error::because`。<br>
在`pingora_error::Result`上调用`or_err(Self,ErrorType,&str)`，返回`because`函数生成的新错误，源错误包装到新错误的cause字段中。

## 重试

错误是可重试的。如果一个错误是可重试的，Pingora代理允许向上游重试。一些错误只允许在[复用的连接](pooling.md)上是可重试的，比如这种情况：尝试复用远端已经丢弃了连接。

默认情况下，新建的`Error`要么是获取它的直接根错误的重试状态，要么在未指定重试状态，被认为是不可重试的。
