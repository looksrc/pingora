# 用户指南

在本指南中，将讲解Pingora比较实用的一些特性、操作、设置。

## 运行Pingora服务器
* [服务启停](start_stop.md)
* [优雅重启和优雅关闭](graceful.md)
* [配置文件](conf.md)
* [守护进程](daemon.md)
* [集成到systemd](systemd.md)
* [恐慌处理](panic.md)
* [错误日志](error_log.md) (翻译需要重新校验)
* [Prometheus](prom.md)

## 构建HTTP代理
* [请求过程: `pingora-proxy` 各阶段及其过滤器](phase.md)
* [连接上游：`Peer`](peer.md)
* [在各阶段共享状态：`CTX`](ctx.md)
* [返回错误](errors.md)
* [例子: 对请求进行控制](modify_filter.md)
* [连接池和连接复用](pooling.md)
* [失败处理和故障转移](failover.md)

## 高级主题 (正在进行中...)
* [Pingora 内部原理](internals.md)
* 使用 BoringSSL
* 用户自定义配置
* Pingora 异步运行时和线程模型
* 后台服务
* 在异步上下文中阻塞
* 跟踪
