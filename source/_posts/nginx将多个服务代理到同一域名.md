title: nginx将多个服务代理到同一域名
date: 2017-06-07 14:38:59
tags:
---

有时候需要将来自不同端口的服务代理到同一个域名上，利用 `nginx` 可以轻易实现这样的需求，记录如下。  

需求如下，两个 `node` 服务分别启动在 `8888` 和 `9999` 端口，然后在访问时需要将 `api.com/version1` 代理到 `127.0.0.1:8888`，将 `api.com/version2` 代理到 `127.0.0.1:9999`。  

利用 `nginx` 的 `location`，搭配 `proxy_pass` 来实现此功能的伪代码如下。  

```nginx
server {
	listen 80;
	server_name api.com;

	location / {
		root /root/www;
		index index.html;
	}

	location ^~ /version1/ {
		proxy_pass http://127.0.0.1:8888/;
	}

	location ^~ /version2/ {
		proxy_pass http://127.0.0.1:9999/;
	}
}
```
此时，访问 `api.com/version1/getUserInfo` 将访问到 `127.0.0.1:8888/getUserInfo`，同理，访问 `api.com/version2/getUserInfo` 将访问到 `127.0.0.1:9999/getUserInfo`。  

**注意**：`proxy_pass` 最后的 `/` 不能丢，否则，`api.com/version1/getUserInfo` 将访问到 `127.0.0.1:8888/version1/getUserInfo`。  

`location` 的匹配规则与优先级的详细可以访问，[nginx配置location总结及rewrite规则写法](http://seanlook.com/2015/05/17/nginx-location-rewrite/?from=https%3A%2F%2Fluoyefe.com)。
