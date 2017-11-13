title: 借助redis实现定时任务
date: 2017-11-13 09:17:18
tags:
---

在 nodejs 中实现定时任务可以使用 [node-schedule](https://github.com/node-schedule/node-schedule) ，但是无法持久化，脚本停止后，不仅任务停止，再次启动也无法恢复未完成的定时任务。

当然也可以使用 crontab 这种方案，但是较为麻烦，其实可以简单的使用 redis 的 pub/sub 功能实现定时任务。

pub/sub 功能简单来说就是，当某个事件被触发时 redis 会向监听此事件的频道推送消息，所以，如果我们监听 redis 中某个 key 的过期事件，即可完成定时任务功能。

## redis 配置

redis 默认未启用 pub/sub 功能，需手动开启。

配置 `redis.conf` 的 `notify-keyspace-events` 字段来开启此功能，其值可以为以下值：

* **K**，表示 `keyspace` 事件，有这个字母表示会往 `__keyspace@[db]__:[event]` 频道推消息

* **E**，表示 `keyevent` 事件，有这个字母表示会往 `__keyevent@[db]__:[event]` 频道推消息

* **g**，表示一些通用指令事件支持，如 `DEL`、`EXPIRE`、`RENAME` 等等

* **$**，表示字符串（String）相关指令的事件支持

* **l**，表示列表（List）相关指令事件支持

* **s**，表示集合（Set）相关指令事件支持

* **h**，哈希（Hash）相关指令事件支持

* **z**，有序集（Sorted Set）相关指令事件支持

* **x**，过期事件，与 **g** 中的 `EXPIRE` 不同的是，**g** 的 `EXPIRE` 是指执行 `EXPIRE key ttl` 这条指令的时候顺便触发的事件，而这里是指那个 `key` 刚好过期的这个时间点触发的事件，即主动与被动的区别

* **e**，驱逐事件，一个 `key` 由于内存上限而被驱逐的时候会触发的事件

* **A**，`g$lshzxe` 的别名，也就是说 `AKE` 的意思就代表了所有的事件

我们要实现定时任务功能，所以需要配置 过期事件，即将 `notify-keyspace-events` 配置为 `Ex`。

配置完成启动 redis 后，可以通过，`redis-cli config get notify-keyspace-events` 命令来验证是否生效。

## redis 事件名规则

`"__keyevent@" + DB_NUMBER + "__:" + EVENT_NAME`

EVENT_NAME: 如 `del` `expire` `rename` 等，具体可看 [redis-evnet-type](http://redisdoc.com/topic/notification.html)

## 业务逻辑

```js
const PublishRedisStore = new redis(); // 发布事件 dbNum: 1
const SubscribeRedisStore = new redis(); // 订阅事件 dbNum: 2

// 订阅者 redis 实例只允许 (P)SUBSCRIBE / (P)UNSUBSCRIBE / PING / QUIT 事件，所以需要两个 redis 实例来完成 pub/sub 功能

SubscribeRedisStore.subscribe('__keyevent@1__:expired', function() {
	// 订阅 redis 数据库 1 的 key expired 事件
});

SubscribeRedisStore.on('message', function(channel, key) {
	// channel 频道名称
	// key (value 已被删除，只能获取到 key 而拿不到 value)
});

// 在 PublishRedisStore 实例中设置某个 3s 后过期的值，key 过期时即可监听到，从而做相应处理
PublishRedisStore.set('textEvent', 'textValue', 'EX', 3);

```

## 其他

除了定时任务功能，借助 pub/sub 还可实现比如 队列消息、循环任务 等功能。

但是一个比较大的限制就是任务时间只能精确到 s，而无法到 ms。
