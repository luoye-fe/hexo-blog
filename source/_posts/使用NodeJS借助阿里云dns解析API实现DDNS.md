title: 使用NodeJS借助阿里云dns解析API实现DDNS
date: 2018-01-03 18:33:50
tags:
---

DDNS，即动态DNS，简单来说就是服务器的 IP 地址经常变动，这个时候需要一个主动上报的服务来更新 DNS 的解析记录，保证域名指向正确的 IP 地址。  

比如在自己家中的树莓派上部署了某些服务，且路由器有公网 IP，由于每次重新拨号，公网 IP 都会变，所以想要根据域名去访问树莓派，必须做 DDNS。

DDNS 可以选择 花生壳，但是花生壳提供的 DDNS 服务无法自定义域名，配置起来也较为麻烦，所以这里用 NodeJS 来实现一个 DDNS 服务。

[源码地址(https://github.com/luoye-fe/aliyun-ddns)](https://github.com/luoye-fe/aliyun-ddns)

## 原理

* 定期获取本机公网 IP

* 比对当前 DNS 解析记录

* 如果不一致，调用阿里云的 API 更新 DNS 记录

## 相关资料

* [阿里云 API](https://help.aliyun.com/document_detail/29739.html)

## 实现

#### 获取公网 IP

访问 `http://ifconfig.me/ip` 获取本机外网 IP，注意需伪造 UA，不然403

#### 阿里云 API 接口鉴权

比较复杂，也比较坑爹，具体实现可以看放出的 git 源码

* 把所有请求参数按顺序序列化

* 把所有请求参数拼接成 `encodeURIComponent(key)=encodeURIComponent(value)&encodeURIComponent(key)=encodeURIComponent(value)` 的形式得到 `signStr`

* 拼接字符串，`[请求方式]&encodeURIComponent('/')&[signStr]`，如 `GET&%2F&[signStr]`

* HMAC SHA1 加密，加密的 key 为 `[AccessKeySecret]&`，注意最后的 `&`

#### 获取当前解析记录

`DescribeSubDomainRecords`

具体看文档

#### 更新或新增解析记录

根据当前解析记录的状态来确定是更新还是新增解析记录

`AddDomainRecord` `UpdateDomainRecord`

#### 定时运行

使用的 [node-schedule](https://github.com/node-schedule/node-schedule)

#### 服务常驻

使用的 [pm2](https://npm.taobao.org/package/pm2)









