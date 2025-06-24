+++
date = '2024-06-21T14:56:25+08:00'
categories = ["linux", "tools", "container"]
title = 'Deploy Coturn Through Docker'
description = "通过docker一键部署coturn(ICE信令服务)"
+++


宿主机准备配置文件
```bash
mkdir -p /opt/docker/coturn/ssl /opt/docker/coturn/compose

# 从github下载配置文件
cd /opt/docker/coturn
wget https://raw.githubusercontent.com/coturn/coturn/refs/heads/master/docker/coturn/turnserver.conf
cp ./turnserver.conf turnserver.conf.default

cd /opt/docker/coturn/ssl
openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout ./privatekey.pem -out ./certificate.pem
```


修改配置文件 `vim /opt/docker/coturn/turnserver.conf`
```bash
# 指定侦听的端口
listening-port=3478

# 云主机内网 IP 地址
listening-ip=xxx.xxx.xxx.xxx

# 云主机的公网 IP 地址
external-ip=xxx.xxx.xxx.xxx

# 这个很重要，如果没有配置这个就无法使用中转服务
# 云主机的公网 IP 地址或域名
realm=xxx.xxx.xxx.xxx

# 访问 STUN/TURN 服务的用户名和密码
user=username:password

# https证书(浏览器webRTC强制需要https)
cert=./ssl/certificate.pem
pkey=./ssl/privatekey.pem

# 内置cli, 可通过telnet连接
cli-ip=0.0.0.0
cli-port=5766
cli-password=CHANGE_ME

# udp端口配置
min-port=49152
max-port=65535
```


创建docker-compose配置 `vim /opt/docker/coturn/composedocker-compose.yaml`
```yaml
services:
  coturn:
    image: coturn/coturn:4.6.2
    container_name: coturn
    restart: unless-stopped
    volumes:
      - /opt/docker/coturn/turnserver.conf:/etc/coturn/turnserver.conf
      - /opt/docker/coturn/ssl:/etc/coturn/ssl
    ports:  # 可直接跳过端口暴露, 使用turnserver.conf文件进行配置
      - "3478:3478"  # 默认
      - "5349:5349"  # tls
      - "5766:5766"  # 内置cli, 可不暴露
      - "49152-65535:49152-65535/udp"
    network_mode: "host"  # 官方推荐直接与宿主机共用端口
```


启动容器
```bash
docker compose up -d
```

```shell
# 放行默认端口 3478(TCP 和 UDP)
sudo ufw route allow 3478
sudo ufw route allow 3478/udp

# 放行 TLS 端口 5349(TCP 和 UDP)
sudo ufw route allow 5349
sudo ufw route allow 5349/udp

# 放行 CLI 端口 5766(如果需要)
sudo ufw route allow 5766
sudo ufw route allow 5766/udp

# 放行 UDP 动态端口范围(49152-65535)
sudo ufw route allow 49152:65535/udp
```

测试地址: `https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice/`

注意: 云服务器需要手动放行防火墙配置文件中指定的端口区间(udp)