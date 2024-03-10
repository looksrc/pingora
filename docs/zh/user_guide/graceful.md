# 优雅重启和优雅关闭

优雅重启、优雅更新、优雅关闭机制，常用于在Pingora服务器发布新版本时防止出现错误和服务中断。

Pingora优雅更新机制可以保证：
* 保证新请求必然会被老的或新的服务进程处理。客户端发起请求后不会被拒绝。
* 请求在宽限时间内都不会被中断.

## 优雅升级

假设程序启动命令为:
```shell
./pingora -d -c pingora.yaml
```

### 步骤 0
在配置文件中设置**升级套接字**文件路径。新旧服务器的套接字必须指向同一个文件。请查看配置手册。
如果新旧程序使用同一个配置文件，且文件没有被修改，则文件路径自然就是相同的。



### 步骤 1
向老实例发送`SIGQUIT信号`。老实例开始将监听套接字发给新实例。
```shell
kill -QUIT $PID
```

### 步骤 2
启动新实例时在命令行指定`-u`/`--upgrade`选项。新实例不会为服务新建监听套接字，而是尝试从老实例接收并继承已建立的套接字并监听。
```shell
./pingora -d -u -c pingora.yaml
```

执行步骤1时，老实例会等待一段时间(让新实例进行初始化并准备接受流量)，时间过后就不再接受新的连接。<br>
一旦执行成功，新实例开始处理新进的连接请求。同时，老实例进入优雅关闭阶段。

<hr>

>
> **译者注**:
> 
> 原文档中步骤1和步骤2是颠倒过来的，但是这样与“快速入门”中的顺序冲突了，而且并不能成功升级。
>
> “快速入门”中实际提供的优雅升级的的总命令为:
> ```shell
> kill -QUIT 17367 && ./pingora -d -u -c pingora.yaml
> ```
> 