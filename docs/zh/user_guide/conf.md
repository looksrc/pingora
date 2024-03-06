# 配置文件

Pingora配置文件格式为yaml。

例子
```yaml
---
version: 1
threads: 2
pid_file: /run/pingora.pid
upgrade_sock: /tmp/pingora_upgrade.sock
user: nobody
group: webusers
```
## 设置
| 字段      | 含义        | 值类型 |
| ------------- |-------------| ----|
| version | 配置文件版本, 例如常数 `1` | number |
| pid_file | pid文件 | string |
| daemon | 是否以守护进程方式运行服务器 | bool |
| error_log | 错误日志输出文件. 默认为`STDERR` | string |
| upgrade_sock | 升级程序时需要使用的套接字文件 | string |
| threads | 每个服务的线程数 | number |
| user | 指定守护进程所属的用户 | string |
| group | 指定守护进程所属的用户组 | string |
| client_bind_to_ipv4 | 服务器连接别人(如上游)时，绑定的源IPv4地址 | list of string |
| client_bind_to_ipv6 | 服务器连接别人(如上游)时，绑定的源IPv6地址| list of string |
| ca_file | 根CA文件 | string |
| work_stealing | 开启任务窃取“运行时”(默认`true`). 参见Pingora运行时一节(进行中...) | bool |
| upstream_keepalive_pool_size | 服务器与上游端点的连接池中保持的总连接数 | number |

## 扩展
无法识别的设置字段会被忽略。但是可以对配置文件进行扩展，以接收用户定义的设置字段。请参考用户自定义设置一节。
