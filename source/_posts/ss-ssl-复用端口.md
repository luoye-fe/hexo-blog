title: shadowsocks/ssl 复用端口
date: 2017-10-25 16:49:02
tags:
---

前提：默认对对 nginx/shadowsocks(后面称 ss) 比较熟

目的：实现 https/ss 共用 443 端口

原因：公知原因 ss 已经会被检测到并封锁，特别是用非常见端口(80/443)，443 ssl 的默认端口来做 ss 服务更加稳定

## 原理

其实很简单，ss 的原理在这不具体讲了，其实作用就相当于代理，把我们的请求通过 ss 这一层从目标服务器拿到数据再给我们。

既然要实现同一个端口访问不同的服务，就是在这两个服务前加一层代理，这层代理可以把 ss 流量转到 ss 的服务，把 web 的流量转到本地的 htps 端口。

## 工具

nginx + shadowsocksR

[shadowsocksR](https://github.com/shadowsocksr-backup/shadowsocksr) 是在 ss 基础上的另一个实现，主要加上了更安全的加密措施等功能，其中对我们今天最重要的就是它的 `redirect` 功能。

shadowsocksR 的 redirect 可以区分流量来源，将 web 请求直接转发到特定端口上去，利用这个就可以实现我们要达到的目的。

## 主体流程

#### nginx

nginx 的安装不表，自行 google

配置好 nginx

只贴下示例了，详细的配置项 google 吧

这里是将 https 的端口放到 `6667` 端口

```
server {
	listen 6667 ssl http2 default_server;
	listen [::]:6667 ssl http2 default_server;
	access_log  /var/log/nginx/www/access.log;
	error_log /var/log/nginx/www/error.log;
	server_name example.com;
	location / {
		root /root/www;
		index index.html index.htm;
	}
	error_page 404 /404.html;

	ssl_certificate "/root/wwwssl/xxx.crt";
	ssl_certificate_key "/root/wwwssl/xxx.key";
}
```

#### shadowsocksR

```
// 下载
git clone https://github.com/shadowsocksr-backup/shadowsocksr.git  
```

```
// 默认分支为 manyuser 主目录为多用户功能，其中的 shadowsocks 子目录为单用户功能  
```

```
// 进入主目录
cd shadowsocksr  
```

```
// 初始化
bash initcfg.sh  
```

```
// 修改配置(看如下示例)
vim user-config.json  
```

```
// 启动 shadowsocksR(-d 参数为后台运行)
python ./shadowsocks/server.py -d start  
```

user-config.json 示例

```json
{
    "server": "0.0.0.0", // ss 服务地址，以本机为 ss 服务端设置为 0.0.0.0
    "server_ipv6": "[::]",
    "server_port": 443, // ss 服务器端口，复用 ssl 所以为 443
    "local_address": "127.0.0.1",
    "local_port": 1080,
    "password": "password", // 密码
    "method": "aes-256-cfb", // 加密方式
    "protocol": "origin", // 默认为 origin
    "protocol_param": "",
    "obfs": "tls1.2_ticket_auth_compatible",
    "obfs_param": "",
    "speed_limit_per_con": 0,
    "speed_limit_per_user": 0,

    "additional_ports" : {}, // only works under multi-user mode
    "additional_ports_only" : false, // only works under multi-user mode
    "timeout": 120,
    "udp_timeout": 60,
    "dns_ipv6": false,
    "connect_verbose_info": 0,
    "redirect": ["*:443#127.0.0.1:6667"], // 重点：将 web 流量转发到本地的 6667 端口
    "fast_open": false
}
```

## 其他

其他端口复用原理一致，shadowsocksR 的 redirect 为数组，支持多个端口转发

