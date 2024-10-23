内网穿透是一种技术手段，用于访问位于防火墙或路由器后面的本地网络（内网）中的设备或服务。通常情况下，内网中的设备无法直接通过公网（互联网）进行访问，从而实现隐私保护和安全性。内网穿透技术的目标是突破这一限制，使外部用户能够通过互联网访问内网中的服务或设备。

市面也有一些内网穿透的服务，包括 `natapp`、`coplar`、`花生壳`等

也可以自己使用云服务器搭建：[fatedier/frp: A fast reverse proxy to help you expose a local server behind a NAT or firewall to the internet.](https://github.com/fatedier/frp)



**服务端**：

frps.toml

```toml
# https://github.com/fatedier/frp/blob/dev/conf/frps_full_example.toml
[common]
# 监听端口
bind_port = 7000
# 面板端口
dashboard_port = 7500
# 登录面板的账号密码（修改成自己的）
dashboard_user = admin
dashboard_pwd = admin
# token = token 如果设定了，需要保持客户端和服务端一致。
```

```yml
# 命令执行 docker-compose -f docker-compose.yml up -d
version: '3.9'
services:
  frps:
    image: fatedier/frps:v0.60.0
    hostname: frps
    container_name: frps
    volumes:
      - "./config/frps.toml:/frps.toml"
    command:
      - "-c"
      - "/frps.toml"
    network_mode: "host"

```

frp 管理后台：http://121.41.226.62:7500/static/#/



**客户端**：

frpc.toml

```yml
# 服务端地址 https://github.com/fatedier/frp/blob/dev/conf/frpc_full_example.toml
serverAddr = "121.41.226.62"
# 服务端配置的bindPort
serverPort = 7000
# token =

[[proxies]]
# 代理应用名称，根据自己需要进行配置
name = "xxx1"
# 代理类型 有tcp\udp\stcp\p2p
type = "tcp"
# 客户端代理应用IP
localIP = "127.0.0.1"
# 客户端代理应用端口
localPort = 8080
# 服务端反向代理端口；提供给外部访问
remotePort = 8080

[[proxies]]
# 代理应用名称，根据自己需要进行配置
name = "xxx2"
# 代理类型 有tcp\udp\stcp\p2p
type = "tcp"
# 客户端代理应用IP
localIP = "127.0.0.1"
# 客户端代理应用端口
localPort = 9001
# 服务端反向代理端口；提供给外部访问
remotePort = 9001

```

```yml
# 命令执行 docker-compose -f docker-compose.yml up -d
version: '3.9'
services:
  frpc:
    image: fatedier/frpc:v0.60.0
    hostname: frpc
    container_name: frpc
    volumes:
      - "./config/frpc.toml:/frpc.toml"
    command:
      - "-c"
      - "/frpc.toml"
    network_mode: "host"

```



本地访问：http://127.0.0.1:8080/api/test

穿透访问：http://121.41.226.62:8080/api/test