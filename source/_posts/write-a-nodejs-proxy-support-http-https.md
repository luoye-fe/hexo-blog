title: write-a-nodejs-proxy-support-http/https
date: 2016-12-02 18:33:41
tags:
---

web 开发的调试过程不可避免的涉及到代理的问题，当然现在很多工具都可以完成这项工作，比如 Win 下的 `Fiddler`，Mac 下的 `Charles`，今天我们来看一下怎么用 `NodeJS` 完成这些代理工作，当然，必须支持 `http/https` 两种常见协议。  

### 代理原理

要想实现代理的目的，就得有一个代理服务的存在，客户端的请求不会直接到达服务器，而是先经过代理服务，然后由代理服务来处理这个请求，不管是原样的转发到目标服务器，还是拦截下来做自定义的回复。

下图来自《HTTP权威指南》，直观的展示了一个代理服务所做的工作：

![proxy](./web_proxy.png.webp)  

实现这个客户端请求转发到代理服务的工具当然可以用系统的代理配置，不过推荐使用 `Chrome` 的一个插件 [Proxy SwitchyOmega](https://chrome.google.com/webstore/detail/proxy-switchyomega/padekgcemlokbadohgkifijomclgjgif).  

### 创建 HTTP 代理服务

了解了代理的原理之后，我们来实现一个简单的代理服务：

```js
const http = require('http');
const url = require('url');
http.createServer()  
    .on('request', (req, res) => {
        // 解析请求参数
        const urlObj = url.parse(req.url);
        const options = {
            hostname: urlObj.hostname,  
            port: urlObj.port || 80,
            path: urlObj.path,
            method: req.method,
            headers: req.headers
        };
        // 新建一个请求到真实服务器
        const pReq = http.request(options, function(curRes) {
            res.writeHead(curRes.statusCode, curRes.headers);
            // 返回给浏览器
            curRes.pipe(res);
        }).on('error', function(e) {
            res.end();
        });
        req.pipe(pReq);
    })
    .listen(8080);
```

上述代码会在本地的 `8080` 端口创建 HTTP 服务，如果配置浏览器的代理为 `127.0.0.1:8080`，这个服务就能接收到浏览器发出的 HTTP 请求，并且从请求中解析相应的参数，新建一个到真实服务器的请求，得到真实服务器响应后，再返回给浏览器，完成一次代理。  

当然，如果你想对转发到本代理服务的请求做拦截并且自定义返回内容，只需要对 `res` 进行操作即可。  

但是，这个代理服务正常运行之后你会发现，HTTPS 的请求无法在本代理服务中捕获，这是为什么呢？  

因为我们现在创建的服务是一个 HTTP 服务，无法处理 HTTPS 请求，那是不是创建一个 HTTPS 的服务就可以了呢？  

答案是肯定的，如果想要代理并且处理浏览器发出的 HTTPS 请求，我们就必须再创建一个 HTTPS 服务。  

创建 HTTPS 服务很简单，重要的是，如何让 HTTPS 请求到达我们的服务器，重新设置浏览器的代理么？并不用这么麻烦。  

### 转发 HTTPS 请求

不管是 HTTP 还是 HTTPS ，都是一种网络应用协议，它最本质的还是一次 TCP 连接，而 TCP 连接我们可以用 CONNECT 方法捕获到，所以有了以下代码：  

```js
const http = require('http');
const net = require('net');
const url = require('url');
http.createServer()
    .on('connect', (req, socket, upgradeHead) => {
        // 利用 `NodeJS` 的 `net` 模块的 `Socket`，处理 CONNECT 请求
        const socketClient = new net.Socket();
        const urlObj = url.parse('https://' + req.url); // CONNECT 只能拿到请求的域名和端口，拼接下再解析

        let realport = urlObj.port || 443;
        let realDomain = urlObj.hostname;
        
        // 转发到真实服务器
        socketClient.connect(realport, realDomain, function() {
            socket.write("HTTP/" + req.httpVersion + " 200 Connection established\r\n\r\n");
        });
        socket.on('data', function(chunk) {
            socketClient.write(chunk);
        });
        socket.on('end', function() {
            socketClient.end();
        });
        socket.on('close', function() {
            socketClient.end();
        });
        socket.on('error', function(err) {
            socketClient.end();
        });
        // 返回给浏览器
        socketClient.on('data', function(chunk) {
            socket.write(chunk);
        });
        socketClient.on('end', function() {
            socket.end();
        });
        socketClient.on('close', function() {
            socket.end();
        });
        socketClient.on('error', function(err) {
            socket.end();
        });
    })
    .listen(8080);
```

上述代码将会在本地的 8080 端口启动一个 HTTP 服务，并且监听 CONNECT 事件并做原样转发，在这个代理服务中我们可以捕获到浏览器的 HTTPS 请求，并且通过我们的代理服务进行向真实服务器的转发工作。  

既然我们已经可以捕获到 HTTPS 请求，那完成对 HTTPS 请求的处理中之后我们就可以得到一个完整的 web 代理服务。  

### 创建 HTTPS 服务

HTTPS 服务的创建 `NodeJS` 为我们提供了 `https` 模块，但是想要创建 HTTPS 服务，必须有 HTTPS 证书才行，HTTPS 证书的颁发国际上有很多机构。作为个人开发者我们可以借助 `openssl` 工具创建自签署的证书。  

具体请谷歌～  

```bash
openssl genrsa -des3 -out private.key 2048 # 生成私钥
openssl req -new -key private.key -out server.csr # 根据开发者提供的信息生成中间文件
openssl x509 -req -in server.csr -out server.crt -outform pem -signkey server.key -days 3650 # 根据中间文件生成证书和公钥
```

有个相应的证书之后我们就可以借此创建 HTTPS 服务：

```js
const https = require('https');
https.createServer({
    key: './server.key',
    cert: './server.crt'
}).listen(9090)
```

上述代码将会在本地的 9090 端口启动 HTTPS 服务。  

### 整合代理服务

能够捕获浏览的 HTTP/HTTPS 请求，也有了 HTTP/HTTPS 服务，我们就可以完整的完成一个代理服务，全部代码如下：  

```js
const http = require('http');
const https = require('https');
const url = require('url');
const net = require('net');

let httpServer = null;
let httpsServer = null;
let httpPort = 8080;
let httpsPort = 9090;

let needProxy = true; // 可以定义代理规则等

httpServer = http.createServer().listen(httpPort);
httpsServer = https.createServer({
    key: './server.key',
    cert: './server.crt'
}).listen(httpsPort);

httpServer.on('request', (req, res) => {
    // 处理 http 请求
    if (needProxy) {
        // 自定义返回内容
        res.write('Proxy server is running.')
        res.end();
    } else {
        // 原样返回
        const urlObj = url.parse(req.url);
        const options = {
            hostname: urlObj.hostname,  
            port: urlObj.port || 80,
            path: urlObj.path,
            method: req.method,
            headers: req.headers
        };
        // 新建一个请求到真实服务器
        const pReq = http.request(options, function(curRes) {
            res.writeHead(curRes.statusCode, curRes.headers);
            curRes.pipe(res);
        }).on('error', function(e) {
            res.end();
        });
        // 返回给浏览器
        req.pipe(pReq);
    }
})

httpsServer.on('request', (req, res) => {
    // 接受到 CONNECT 转发来的 HTTPS 请求，并自定义返回内容
    res.write('Proxy server is running.')
    res.end();
})

httpServer.on('connect', (req, socket, upgradeHead) => {
    // 利用 `NodeJS` 的 `net` 模块的 `Socket`，处理 CONNECT 请求
    const socketClient = new net.Socket();
    const urlObj = url.parse('https://' + req.url); // CONNECT 只能拿到请求的域名和端口，拼接下再解析

    let realport = urlObj.port || 443;
    let realDomain = urlObj.hostname;
    
    if (needProxy) {
        // 转发到本地 HTTPS 服务
        socketClient.connect('127.0.0.1', httpsPort, function() {
            socket.write("HTTP/" + req.httpVersion + " 200 Connection established\r\n\r\n");
        });
    } else {
        // 转发到真实服务器
        socketClient.connect(realport, realDomain, function() {
            socket.write("HTTP/" + req.httpVersion + " 200 Connection established\r\n\r\n");
        });
    }
    
    socket.on('data', function(chunk) {
        socketClient.write(chunk);
    });
    socket.on('end', function() {
        socketClient.end();
    });
    socket.on('close', function() {
        socketClient.end();
    });
    socket.on('error', function(err) {
        socketClient.end();
    });
    // 返回给浏览器
    socketClient.on('data', function(chunk) {
        socket.write(chunk);
    });
    socketClient.on('end', function() {
        socket.end();
    });
    socketClient.on('close', function() {
        socket.end();
    });
    socketClient.on('error', function(err) {
        socket.end();
    });
})
```

不到 100 行代码，一个简单的 proxy 服务器已经搭建完成，支持 HTTP/HTTPS 代理，在这个基础上，完全可以进行再开发，比如代理规则的扩充，查看浏览器请求流，具体应用等。  

本文就写到这里，欢迎评论留言。  
