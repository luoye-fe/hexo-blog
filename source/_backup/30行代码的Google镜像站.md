title: 30行代码的Google镜像站
date: 2018-01-16 10:45:28
tags:
---

#### **DEMO: [SO FREEDOM](http://so.luoyefe.com/)**

#### **源码地址: [https://github.com/luoye-fe/so/](https://github.com/luoye-fe/so/)**

又是一个充满噱头的标题 = =。

思路是在研究 `NodeJS` 代理时想到的小 trick，说是 30 行，其实最重要的只有一行，即：

```js
req.pipe(request(URL)).res;
```

将客户端来的请求直接转发到相应的 URL 上，实现代理，基于这个原理，可以很容易的实现各种网站的镜像站点。

服务端代码加注释如下：

```js
const fs = require('fs');
const URL = require('url');
const http = require('http');
const request = require('request');

const port = 9901;
const homeHTML = fs.readFileSync('./home.html', 'utf-8'); // 自定义的首页内容

function requestInstance(url, ua) {
	return request({
		url,
		headers: { 'User-Agent': ua }
	})
}

const Server = http.createServer((req, res) => {
	const url = URL.parse(req.url, true);
	if (url.pathname === '/') {
		// 自定义首页
		res.writeHeader('200', { 'Contetn-Type': 'text/html' });
		fs.createReadStream('./home.html').pipe(res);
	} else if (url.pathname === '/bg') {
		// 每日必应壁纸加速（给首页用，具体可看源码内逻辑）
		req.pipe(requestInstance(`https://bing.ioliu.cn/v1?${url.search}`, req.headers['user-agent'])).pipe(res);
	} else {
		// 这个就是最重要的一步了，将客户端来的请求，按照规则直接代理到 `google.com` 下
		// 这样客户端拿到的内容也是谷歌返回的内容了
		// 因为请求是服务端转发到谷歌，所以翻墙这一步也就免了
		req.pipe(requestInstance(`https://www.google.com/${url.path}`, req.headers['user-agent'])).pipe(res);
	}
});

// 启动服务
Server.listen(port, () => {
	console.log(`Server on port ${port}`);
});

```

几行代码，完整可用的免翻墙 Google 搜索就完成了。

按这个思路来，任何网站都可以用这样的思路来玩耍，比如 `po**hub.com` ??

