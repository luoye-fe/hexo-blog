title: JS 新特性：Temporal
date: 2020-09-21 20:40:00
tags:
---
## 前言

近日（2020.9.16）， `Moment` 团队发布[声明](https://momentjs.com/docs/#/-project-status/recommendations/)，宣布 `Moment` 将进入**维护状态**，除非必要，将不再更新：

- 未来不会新增任何新特性
- 不会改成 `immutable` 
- 不会做 `tree shakeing` ，也不会解决包体积问题
- 不会更新 v3 版本
- 不会修复 bug 或非预期行为，特别是长期以来的已知问题

同时，声明中着重强调了新项目不要再使用 `Moment` ，并且给出了充分的理由，以及替代方案， 其中提到了一个正处于 stage2 状态的 js 新特性 `Temporal` ，虽然离生产还有很长的路，但是不妨提前学习下这个未来可以取代所有第三方时间库的新特性。

学习 `Temporal` 之前，先来看下从 JS 诞生之初就存在的 `Date` 有什么问题，为何重头设计一个全新的事件对象：

- **不支持时区**：`Date` 对象无法修改时区，永远以当前系统的时区为准，如果需要计算不同时区的时间，只能手动根据时区之间的时间差手动计算

- **时间格式化不可靠**：这个很好理解，应该没有一个 JSER 会觉得原生的 `Date` 对象格式化成 `YYYY-MM-DD` 是个简单操作

- **`Date` 对象是 `mutable` 的**：与 `String` 对象的所有方法都返回一个新对象，不会修改原始字符串对象不同，对 `Date` 对象的操作都会修改原始对象，导致结果难以保证

- **夏令时行为诡异**：中国有一段时间（1986-1991）实行过夏令时，所以这个时间段的时间在不同的浏览器上得到的时间戳不一致，并且没有直接判断是夏令时的 API

- **时间计算 API 难用至极**：需要计算两个时间的间隔或者时间增减时，API 及其难用

- **等等**

所以 TC39 工作组提出了遵循 [ISO_8601 标准](https://zh.wikipedia.org/zh-hans/ISO_8601) 的全新时间对象 `Temporal` 来解决这些问题，全新的对象将着眼于以下几个方面：

- 提供简单易用的 API
- 提供 `immutable` 对象
- 严格的时间字符串格式
- 支持非公历、支持时区

详细的 API 可以查看 [TC39文档](https://tc39.es/proposal-temporal/docs/)，接下来我们用实际的案例感受下 `Temporal` 的强大（在 `Temporal` 文档页面，已集成 polyfill，可在控制台直接调用）。

## 实践

### 获取/构造时间

相较于 `Date`，使用 `Temporal` 获取/构造时间有以下优点：

- 最大精度为纳秒，可以对时间进行更精准的计算（为了配合纳秒， `Temporal` 以 `bigInit` 为基础参数） 
- 标准的时间构造方法
- 完整的时间静态属性

> `bigInit` 是 JS 新特性之一，简单来讲，在 `number` 后跟上 `n` 即为一个 `bigInit` 变量，比如 `0n`  `12345678900987654321n` ，此处不具体展开。

```javascript
// 当前系统时间
const now = Temporal.now.dateTime().toString(); // -> 2020-09-18T11:49:34.076618285

// 忽略时区的绝对时间
const now = Temporal.now.absolute().toString(); // -> 2020-09-18T03:49:34.871974076Z

// 构造距 1970-1-1 xx 纳秒的 Temporal 对象
const now = new Temporal.Absolute(0n).toString(); // -> 1970-01-01T00:00Z

// 根据 年 月 日 时 分 秒 毫秒 微妙 纳秒 构造时间 !!!
const now = new Temporal.DateTime(2020, 9, 20, 12, 15, 32, 666, 233, 999).toString(); // -> 2020-09-20T12:15:32.666233999

// 更全的时间静态属性
const now = Temporal.now.dateTime();
/* ⬇️⬇️⬇️
{
  year: 2020, // 年
  month: 9, // 月
  day: 21, // 日
  hour: 17, // 时
  minute: 18, // 分
  second: 7, // 秒
  millisecond: 771, // 毫秒
  microsecond: 785, // 微妙
  nanosecond: 972, // 纳秒
  
  dayOfWeek: 1, // 全周的第几天（从1开始）
  dayOfYear: 265, // 全年的第几天
  weekOfYear: 39, // 全年的第几周
  
  daysInWeek: 7, // 当前时间所在周的天数
  daysInMonth: 30, // 当前时间所在月的天数
  daysInYear: 366, // 当前时间所在年的天数
  monthsInYear: 12, // 当前时间所在年的月数

  isLeapYear: true, // 是否是闰年
}
*/
```

### 时间计算

原生时间对象 `Date` 缺少时间计算相关 API，如果需要在当前时间基础上新增或减少天数得到另一个时间，只能通过先获取天数，再进行加减，再合并成完整时间，更要考虑跨天、跨月、跨年，甚至闰年、夏令时的影响，极其难用。

相比较而言， `moment.js` `day.js` 等第三方库提供了便捷的时间计算 API，我们来做个简单比较

```javascript
// day.js / moment.js
const futureTime = dayjs().add(7, 'day').toString(); // -> Mon, 28 Sep 2020 09:33:31 GMT
const futureTime = dayjs().subtract(7, 'd').toString(); // -> Mon, 14 Sep 2020 09:39:45 GMT

// Temporal
const futureTime = Temporal.now.dateTime().plus({ days: 7 }).toString(); // -> Mon, 28 Sep 2020 09:33:31 GMT
const futureTime = Temporal.now.dateTime().minus(new Temporal.Duration('P7D')).toString(); // -> 2020-09-21T17:40:17.539198751

// P7D 为 Temporal 中构造段时间的语法，简介如下
// P1Y1M1DT1H1M1.1S -> 一年一月一天一小时一分一百毫秒
// P1M -> 一个月
// PT1M -> 一分钟
```

### 时间展示

对大部分场景来说，展示时间是采用各种第三方库的首要原因，这样非常痛点的能力在 `Temporal` 是怎样使用的呢？

#### 时间格式化

很遗憾，目前 `Temporal` 未提供与 `day.js` 类似的占位符格式化 API。不过根据上文提到的时间静态属性，结合模板字符串，与原生 `Date` 相比，格式化的工作量也将大大降低。根据这些静态属性，实现一个类似三方库的 API，工作量也不会太大。

#### 时间偏差

计算两个时间之间的差值，展示时间差也是一个经常遇到的场景， `Temporal` 提供的 API 与 `day.js` 相比更具灵活性，差值单位甚至可以多单位组合。

```javascript
// day.js / moment.js
dayjs('2019-01-25').diff('2017-06-05', 'month'); // -> 19
dayjs('2019-01-25').diff('2017-06-05', 'm'); // -> 19

// Temporal
Temporal.DateTime.from('2019-01-25').difference(Temporal.DateTime.from('2017-06-05')).toString(); // -> P599D = 599天
Temporal.DateTime.from('2019-01-25').difference(Temporal.DateTime.from('2017-06-05'), {
  largestUnit: 'years',
  smallestUnit: 'seconds'
}).toString(); // -> P1Y7M20D = 一年七个月二十天

```

### 时区支持

与 `Date` 不同， `Temporal` 将天然支持时区。 `Temporal` 的时间默认无时区信息，所以需要在不同时区转换时间信息时，将用到时区相关 API。

```javascript
// 构造中国东八区时区
const tz = new Temporal.TimeZone('Asia/Shanghai'); // 时区参数支持 IANA 名称，支持 UTC等 https://en.wikipedia.org/wiki/List_of_tz_database_time_zones

// 获取绝对时间
const absoluteNow = Temporal.now.absolute(); // -> 2020-09-21T12:02:50.041733410

// 时间转换
const now = tz.getDateTimeFor(Temporal.now.absolute()); // -> 2020-09-21T20:02:50.041733410
```

### 历法支持

在上文提到的时间静态属性中，细心的朋友可能已经发现有是否是闰年这个属性，其实这就是历法在起作用。

时间定义中，历法决定了了一年中有多少天，一年有多少月，一个月有多少天，所以能够灵活而准确的采用不同历法在时间计算中相当重要。

目前世界上最常用的历法是 [Gregorian calendar](https://en.wikipedia.org/wiki/Gregorian_calendar)，在此基础上演变出来的时间标准 ISO_8601， `Temporal` 就是采用此种标准。

以 [Hebrew calendar](https://en.wikipedia.org/wiki/Hebrew_calendar)（犹太历）为例，犹太历根据大小月，每个月有29或30天，八月九月是润月，根据常年缺年满年的不同，有不同的天数，以犹太历为例：

```javascript
// 获取犹太历 5756年3月14号 的绝对时间
const dateWithCalendar = Temporal.DateTime.from({
    year: 5756,
  month: 3,
  day: 14,
  hour: 3,
  minute: 24,
  second: 30,
  calendar: new Temporal.Calendar('hebrew'),
}); // -> 1995-12-07T03:24:30[c=hebrew]
```

### 其他

其他方面， `Temporal` 相较于 `Date` 也有非常多的新特性，比如本地化、验证、比较等，期待 `Temporal` 尽早进入 stage3 甚至 stage4 阶段，彻底替代所有第三方时间库。 
