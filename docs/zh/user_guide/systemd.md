# 集成到systemd

虽然Pingora服务器并不依赖于systemd，但是可以轻易的将其集成到systemd服务中。

```ini
[Service]
Type=forking
PIDFile=/run/pingora.pid
ExecStart=/bin/pingora -d -c /etc/pingora.conf
ExecReload=kill -QUIT $MAINPID
ExecReload=/bin/pingora -u -d -c /etc/pingora.conf
```

这段systemd配置集成了Pingora的优雅升级操作。如要要优雅升级程序二进制文件，只需要执行`systemctl reload pingora.service`。
