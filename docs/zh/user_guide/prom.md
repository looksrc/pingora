# Prometheus

Pingora 有一个内置的 Prometheus HTTP 指标服务器，用于抓取指标。

```rust
...
let mut prometheus_service_http = Service::prometheus_http_service();
prometheus_service_http.add_tcp("0.0.0.0:1234");
my_server.add_service(prometheus_service_http);
my_server.run_forever();
```

最简单的使用方式是使用[静态指标](https://docs.rs/prometheus/latest/prometheus/#static-metrics)

```rust
static MY_COUNTER: Lazy<IntGauge> = Lazy::new(|| {
    register_int_gauge!("my_counter", "my counter").unwrap()
});

```

静态指标会自动出现在 Prometheus 指标端点上, <http://address:port/metrics>。
