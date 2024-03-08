# 集成到systemd

虽然Pingora服务器并不依赖于systemd，但是可以轻易的将其集成到systemd中。

## 程序准备
* 可执行程序: /bin/pingora
* 配置文件：/etc/pingora.conf

## 服务添加
服务路径: /usr/lib/systemd/system/pingora.service

文件内容: 
```ini
[Unit]
Discription=Pingora Server

[Service]
Type=forking
PIDFile=/run/pingora.pid
ExecStart=/bin/pingora -d -c /etc/pingora.conf
ExecReload=kill -QUIT $MAINPID
ExecReload=/bin/pingora -u -d -c /etc/pingora.conf
ExecStop=kill -TERM $MAINPID
```
这段systemd配置集成了Pingora的优雅升级操作。如要要优雅升级程序二进制文件，只需要执行`systemctl reload pingora.service`。

重载systemd，使新加的服务生效：
```shell
systemctl daemon-reload
```

## 服务操作


启动 Pingora：
```shell
systemctl start pingora
```

优雅升级或重载 Pingora(优雅升级前要人工先替换掉老的二进制文件)：
```shell
systemctl reload pingora
```

优雅关闭 Pingora，时间略长：
```shell
systemctl stop pingora
```