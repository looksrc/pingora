# 快速入门: 负载均衡器

## 介绍

此快速入门介绍如何利用`pingora`和`pingora-proxy`构建一个简单的负载均衡器。

要实现的负载均衡器：
- 两个上游：https://1.1.1.1, https://1.0.0.1
- 负载策略：轮询(round-robin)

## 构建负载均衡器

为负载均衡器创建一个新的`cargo`工程，起名为`load_balancer`。

```
cargo new load_balancer
```

### 引入Pingora库和基本的依赖

在工程配置文件`cargo.toml`中添加下面的依赖:
```
async-trait="0.1"
pingora = { version = "0.1", features = [ "lb" ] }
```

### 创建Pingora服务器

首先，创建一个Pingora服务器。Pingora的`Server`是一个进程，可以承载一个或多个服务。<br>
`Server`主要负责解析配置文件和CLI参数、守护进程化、信号处理、优雅重载或关闭。

比较推荐的做法是在`main`函数中初始化`Server`，然后通过`run_forever`孵化所有的运行时线程，阻塞`main`线程直到服务器准备关闭。

```rust
use async_trait::async_trait;
use pingora::prelude::*;
use std::sync::Arc;

fn main() {
    let mut my_server = Server::new(None).unwrap();
    my_server.bootstrap();
    my_server.run_forever();
}
```

这段代码可以通过编译，但是没有什么实际作用。

### 创建负载均衡代理
下一步，创建一个负载均衡器。负载均衡持有所有上游IP的静态列表。<br>
`pingora-load-balancing`库已经为`LoadBalancer`结构体提供了常用的负载算法如轮询、哈希，直接使用即可。如果需要更复杂的或者自定义的负载策略可以自行实现。<br>

```rust
pub struct LB(Arc<LoadBalancer<RoundRobin>>);
```

为了将服务器变成一个代理，需要为`LB`类型实现`ProxyHttp`特质。

为对象实现`ProxyHttp`特质本质上是定义了请求在代理中的处理逻辑。在`ProxyHttp`中唯一必须的方法是`upstream_peer()`，此方法用于决定当前请求需要转发给哪个后端，方法的返回值即为最终确定的后端。

在`upstream_peer()`的函数体中，通过调用`LoadBalancer`的`select()`方法从上游列表中轮询选择目标后端。<br>
在本例子中，使用HTTPS连接到后端，因此在构造[`Peer`](peer.md)对象时还需要传入`use_tls`参数(`true`)以及sni参数(`one.one.one.one`)。

```rust
#[async_trait]
impl ProxyHttp for LB {

    /// 本例子不需要上下文存储
    type CTX = ();
    fn new_ctx(&self) -> () {
        ()
    }

    async fn upstream_peer(&self, _session: &mut Session, _ctx: &mut ()) -> Result<Box<HttpPeer>> {
        let upstream = self.0
            .select(b"", 256) // 哈希不影响轮询
            .unwrap();

        println!("upstream peer is: {:upstream?}");

        // 将 SNI 设为 one.one.one.one
        let peer = Box::new(HttpPeer::new(upstream, true, "one.one.one.one".to_string()));
        Ok(peer)
    }
}
```

为了让1.1.1.1后端接受请求，必须为请求设置主机头Host。`ProxyHttp`的`upstream_request_filter()`方法可以在和后端成功建立连接后、请求头被发送之前修改发送给后端的HTTP请求头。

```rust
impl ProxyHttp for LB {
    // ...
    async fn upstream_request_filter(
        &self,
        _session: &mut Session,
        upstream_request: &mut RequestHeader,
        _ctx: &mut Self::CTX,
    ) -> Result<()> {
        upstream_request.insert_header("Host", "one.one.one.one").unwrap();
        Ok(())
    }
}
```


### 创建Pingora代理服务
下一步，利用上面的负载均衡器创建一个代理服务。

Pingora的`Service`监听一个或多个端点(TCP 或 Unix domain socket)。连接建立成功后，`Service`将连接交给它的“应用程序”，`pingora-proxy`就是一种应用程序，用于将HTTP请求代理到上面配置的后端。

在本例子中，创建了一个`LB`实例，带有`1.1.1.1:443` 和 `1.0.0.1:443`两个后端。
通过函数`http_proxy_service()`创建一个携带`LB`实例的代理`Service`，然后再将`Service`注册到`Server`中。

```rust
fn main() {
    let mut my_server = Server::new(None).unwrap();
    my_server.bootstrap();

    let upstreams =
        LoadBalancer::try_from_iter(["1.1.1.1:443", "1.0.0.1:443"]).unwrap();

    let mut lb = http_proxy_service(&my_server.configuration, LB(Arc::new(upstreams)));
        lb.add_tcp("0.0.0.0:6188");

    my_server.add_service(lb);

    my_server.run_forever();
}
```

### 运行

现在，我们已经将负载均衡添加到了服务中，可以用下面的命令运行程序

```cargo run```

通过curl命令向服务器发送测试请求:
```
curl 127.0.0.1:6188 -svo /dev/null
```

也可以通过浏览器访问地址 [http://localhost:6188](http://localhost:6188)

下面的输出表明负载均衡已经开始工作，在两个后端之间做负载：
```
upstream peer is: Backend { addr: Inet(1.0.0.1:443), weight: 1 }
upstream peer is: Backend { addr: Inet(1.1.1.1:443), weight: 1 }
upstream peer is: Backend { addr: Inet(1.0.0.1:443), weight: 1 }
upstream peer is: Backend { addr: Inet(1.1.1.1:443), weight: 1 }
upstream peer is: Backend { addr: Inet(1.0.0.1:443), weight: 1 }
...
```

很好！此时我们就有一个可以工作的负载均衡器了。
但是这只是一个非常基本的负载均衡器，下一部分讲解如何利用一些Pingora内置的工具让它变得更健壮。

## 添加功能

Pingora提供了一些实用的特性，只需要几行代码就可以启用并配置这些特性。
例如负载均衡后端健康检查、不停机更新代理程序的二进制文件等等。

### 对端健康检查

为了让负载均衡器更可靠，我们需要为上游添加健康检查。一旦其中某个后端挂了，迅速停止向此后端分发流量。

为负载均衡添加一个不健康的后端，用来观察混入不健康节点后负载均衡的表现。

```rust
fn main() {
    // ...
    let upstreams =
        LoadBalancer::try_from_iter(["1.1.1.1:443", "1.0.0.1:443", "127.0.0.1:343"]).unwrap();
    // ...
}
```

执行`cargo run`命令启动负载均衡器，并使用curl命令进行测试：

```
curl 127.0.0.1:6188 -svo /dev/null
```

可以看到，每三个请求中就有一个报`502: Bad Gateway`，这是因为负载均衡器严格的执行`RoundRobin`负载策略，没有考虑到对端是否还处于健康状态。
我们可以通过添加基本的健康检查服务来修复这个问题。

```rust
fn main() {
    let mut my_server = Server::new(None).unwrap();
    my_server.bootstrap();

    // 注意，此时upstreams变量必须声明为`mut`
    let mut upstreams =
        LoadBalancer::try_from_iter(["1.1.1.1:443", "1.0.0.1:443", "127.0.0.1:343"]).unwrap();

    let hc = TcpHealthCheck::new();
    upstreams.set_health_check(hc);
    upstreams.health_check_frequency = Some(std::time::Duration::from_secs(1));

    let background = background_service("health check", upstreams);
    let upstreams = background.task();

    // `upstreams` 无需再包装进Arc中了，因为它已经在Arc中
    let mut lb = http_proxy_service(&my_server.configuration, LB(upstreams));
    lb.add_tcp("0.0.0.0:6188");

    my_server.add_service(background);

    my_server.add_service(lb);
    my_server.run_forever();
}
```

此时，再运行并测试负载均衡器，可以看到所有的请求都是成功的，再也不会被路由到不健康的后端节点上。
基于我们使用的配置，如果节点重新变为健康状态，会在1秒后重新被纳入轮询列表中。

### 命令行选项

`Server`提供了大量的内置功能，只需要修改一行代码就能将他们利用起来。

```rust
fn main() {
    let mut my_server = Server::new(Some(Opt::default())).unwrap();
    ...
}
```

修改后，通过命令行传给负载均衡的参数会被Pingora读取。可通过以下命令查看所有支持的参数：

```
cargo run -- -h
```

我们会看到一个帮助菜单，显示当前可用的参数列表。
在下一节将利用这些参数为负载均衡器轻松做一些配置。

### 守护进程

传入`-d` or `--daemon`参数让程序以守护进程形式运行。

```
cargo run -- -d
```

发送`SIGTERM`信号给负载均衡器，可以停止服务并优雅关闭，收到信号后服务不再接受新的请求，但是在退出前会尝试完成所有正在进行的请求。
```
pkill -SIGTERM load_balancer
```
 (`SIGTERM` 是 `pkill`命令的默认信号.)

### 服务器配置

Pingora的配置文件可以定义如何运行我们的运行负载均衡程序。<br>
下面是一个示例文件，定义了程序的线程数、pid文件的位置、错误日志文件的位置、升级协作套接字的位置(后面介绍)。<br>
将下面的内容复制到`load_balancer`工程目录下的`conf.yaml`文件中。

```yaml
---
version: 1
threads: 2
pid_file: /tmp/load_balancer.pid
error_log: /tmp/load_balancer_err.log
upgrade_sock: /tmp/load_balancer.sock
```

使用此配置文件：
```
RUST_LOG=INFO cargo run -- -c conf.yaml -d
```
`RUST_LOG=INFO` 让程序可以打印错误日志.

现在可以观察到程序的pid文件：
```
 cat /tmp/load_balancer.pid
```

### 优雅升级
(仅对Linux)

假设我们已经更改了负载均衡器的代码，重新编译了二进制文件。现在想要将守护进程更新为新程序。

如果简单的停止老的程序，然后启动新的程序，一些已经到达但是未完成的请求会丢失。还好Pingora提供了可以优雅的更新程序的方式。

步骤为，首先向老的进程发送`SIGQUIT`信号，然后传入`-u` \ `--upgrade`选项启动新程序。

```
pkill -SIGQUIT load_balancer &&\
RUST_LOG=INFO cargo run -- -c conf.yaml -d -u
```

在这个过程中，老的进程会等待并将它正在监听的套接字传给新的进程。然后老进程处理完所有请求后退出。

从客户端的视角来看，由于所有的监听套接字并未关闭，所以服务进程看起来一直在运行从未停止过。

## 完整代码

本例子的完整代码在本仓库中：

- [pingora-proxy/examples/load_balancer.rs](../../pingora-proxy/examples/load_balancer.rs)

此外还提供了别的一些有用的例子：

- [pingora-proxy/examples/](../../pingora-proxy/examples/)
- [pingora/examples](../../pingora/examples/)