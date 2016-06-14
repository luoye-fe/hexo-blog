title: js中的异步流程控制--Promise/Generator/Async/Await
date: 2016-06-09 08:05:14
tags:
---

> 长文预警 ～

异步I/O、事件驱动使JS这个单线程语言在不阻塞的情况下可以并行的执行很多任务，这带来了性能的极大提升，并且更加符合人们的自然认识（烧一壶水，期间你肯定不会等着水烧开再去做别的事，异步才是正常的啊！）。然而异步风格也给流程控制，错误处理带来了更多的麻烦。

## 异步流程控制的解决方案

### 一、回调

回调是JS的基础，函数可以作为参数传递并在恰当的时机执行，比如有下面的三个函数：   

```js
f1();
f2();
f3();
```

如果 `f1` 中存在异步操作，比如 `ajax` 请求，并且 `f2` 需要在 `f1` 执行完毕之后执行，那么可以使用回调的方式改写函数，如下：  

```js
var f1 = function(cb){
	$.ajax({
		url: '...',
		type: 'get',
		success: function(){
			cb && cb();
		}
	})
}

var f2 = function(){
  	// do something after f1 complete ...
}

var f3 = function(){
  	// do something else
}

f1(f2);
f3();
```

使用这种方式， `f1` 的异步操作，不会阻碍程序的运行，并且可以很方便的控制函数的执行过程，显然，我要说但是了。如果你看到下面的代码，估计你不会觉得回调有那么美好了。  

```js
f1(function(err, data){
	f2(function(err, data){
		f3(function(err, data){
			f4(function(err, data){
				f5(function(err, data){
		  			f6(function(err, data){
		    			// maybe more ...
		  			})
				})
			})
		})
	})
})
```

WTF?!

可以看出，回调的缺点很明显，各个函数高度耦合，代码结构混乱，`debug` 困难，等等。  

### 二、事件监听（观察者模式）

另一种解决异步流程控制的方法是采用事件监听的机制，某个事件的触发不再以某个时机为界限，而是取决于某个事件是否触发。  

```js
var f1 = function(){
	setTimeout(function(){
		Event.trigger('loaded', argvs);
	}, 2000)
}

Event.on('loaded', function(argvs){
	// do something ...
})

f1();
```

唔，很美好的解决方案，但是观察者模式的缺点在其中也体现的很明显，事件的监听和触发散落在不同的地方，程序趋于复杂之后，`Event` 机制的复杂度也极大提高，明显这不是我们追求的。  

### 三、异步流程控制库

为了优雅的解决异步流程控制的问题，伟大的猿们前赴后继，产出了很多方案，造就了不少优秀的库，包括但不限于 `q` `co` `async` 等。  

这些库的具体实现或使用方式不在本文的谈论范围，暂时跳过。  

### 四、新标准、新未来

> 重点来了！

现在已经是2016年了，`ES` 的标准一代快过一代，有了 `bable` 这样的工具，甚至 `ES7` 都不再是不可触及的 `feture`了，新的标准当然对异步控制做出了很多努力，让我们一个一个来看。  

#### 1、Promise

[Promise - MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise)

所谓的 `Promise` ，就是一个特殊的用于传递异步信息的对象，它代表一个未完成的但是将会完成的操作。也就是说，`Promise` 代表了某个未来才会知道结果的事件（通常是一个异步操作），并且为这个异步事件提供统一的 `API`，能够让使用者准确的控制异步操作的每一个流程。  
  
###### a. 基本理解

* 一个 `Promise` 对象，存在三种状态， `pending(进行中)`、`resolve(已完成)`、`reject(已失败)`。一个异步操作的开始，对应着 `Promise` 的 `pending` 状态，异步操作的结束，对应着另两种状态，当异步操作成功时，对应着 `resolve`状态，失败时对应着 `reject`状态。

* `Promise` 的状态如果发生改变，就不能再被更改，并且，只能由 `pending` 向另外两种状态转变，不能逆，也不能 `resolve` 和 `reject` 互相转化。  

###### b. 基本 API

* Promise.resolve() 
* Promise.reject()
* Promise.prototype.then()
* Promise.prototype.catch()
* Promise.all()
* Promise.race()

###### c. 详解

* 创建 `Promise` 实例
	
	```js
	var promise = new Promise(function(resolve, reject){
		// async operation ...
		if( /* async operation success */ ){
			resolve(value);
		}else{
			reject(err);
		}
	})
	```

	构造函数 `Promise` 接受一个函数作为参数，这个函数又有两个类型为方法的参数，`resolve` 、 `reject`。`resolve` 方法用来将 `promise` 从 `pending` 状态转换到 `resolve` 状态，并且将异步操作成功后返回的内容传递出去，`reject` 方法用来将 `promise` 从 `pending` 状态转换到 `reject` 状态，在异步操作失败时调用，并传递错误信息。  

* 调用

	`Promise` 实例创建后，可以调用 `then` 方法，处理异步操作成功或失败的状态。  

	`then` 方法接受两个函数参数，第一个即为创建 `Promise` 实例时的 `resolve` 函数，第二个则为创建 `Promise` 实例时的 `reject` 函数，用来分别处理异步操作成功，或失败的后续操作。当然，第二个用来处理失败的参数为可选参数。  

	```js
	promise.then(function(value){
		// async operation success
	}, function(err){
		// async operation failed
	})
	```

* 示例1: `sleep` 函数

	在很多编程语言中，都有着 `sleep` 函数，延迟程序执行，`javascript` 中可以用 `setTimeout` 完成操作的延迟执行，但是还是需要使用回调的方式，现在让我们用 `Promise` 来实现。  

	```js
	var sleep = function(ms){
		return new Promise(function(resolve, reject){
			setTimeout(resolve, ms);
		})
	}

	// 休眠1000ms后执行
	sleep(1000).then(function(){
		console.log('1000s gone')
	})
	```

	一个简单的休眠函数就完成了，调用更加方便，也更加直观。  

* 示例2: 异步 `Ajax` 请求

	```js
	// 封装下原生 XMLHttpRequest 操作
	var ajaxExample = function(params){
		var promise = new Promise(function(resolve, reject){
			var client = new XMLHttpRequest();
			client.open(params.type, params.url);
			client.onreadystatechange = handler;
			client.send();

			function handler() {
				if (this.readyState !== 4) {
					return;
				}
				if (this.status === 200) {
					resolve(this.response);
				} else {
					reject(new Error(this.statusText));
				}
			};
		})
		return promise;
	}

	// 调用
	ajaxExample({
		url: '/test',
		type: 'GET',
		data: {
			page: 2
		}
	}).then(function(res){
		console.log(res)
	}, function(err){
		console.log(err)
	})
	```

* `Promise.prototype.then()`
	
	上面两个简单的示例，展示了 `Promise` 的基本使用方法，让我们再来看看具体的 `API`。  

	`then` 方法除了用于处理 `Promise` 实例的成功或失败操作，还会返回一个新的 `Promise` 实例，并且将返回值传递给下一层 `then` 方法，即：  

	```js
	sleep(1000)
		.then(function(){
			console.log('1000s gone')
			return '123'
		})
		.then(function(val){
			console.log(val) // 123
		})
	```

	这样来看，曾经使用多层嵌套的回调来控制异步流程的代码终于可以下岗了。

* `Promise.prototype.catch()`

	在 `then` 方法中，第二个参数可以对当前 `Promise` 中的错误进行处理，为了统一的错误处理，`Promise` 也为我们提供了一个更加方便的错误处理方式。  
	
	当一个 `Promise` 实例转变为 `reject` 状态的时候，会调用 `catch` 中的回调函数，并且把首次 `reject` 的错误传递进去。  

	```js
	var promise = new Promise(function(resolve, reject){
		reject('error test');
	})
	promise.catch(function(err){
		console.log(err); // error test
	})
	```

	`catch` 能够捕获 `reject` 主动抛出的错误，同样也能捕获 `Promise` 运行中的错误。  
	
	```js
	var promise = new Promise(function(resolve, reject){
		throw new Error('error test');
	})
	promise.catch(function(err){
		console.log(err); // Error: error test(…)
	})
	```

	`catch` 捕获错误时具有冒泡属性，即在最后调用 `catch` 时，能够捕获到此前所有 `Promise` 中的错误。  

	```js
	ajaxExample({
		url: '/test',
		type: 'GET',
		data: {
			page: 2
		}
	}).then(function(res){
		console.log(res)
	}).catch(function(err){
		// 处理前两个 Promise 中的错误
	})
	```

	上面的示例中，最后的 `catch` 方法能够捕获到前两个 `Promise` 中任意一个产生的错误。  

* `Promise.all()`

	`Promise.all` 方法用于将多个Promise实例，包装成一个新的Promise实例。  

	```js
	var allPromise = Promise.all([p1, p2, p3])
	```

	`Promise.all` 接受一个由多个 `Promise` 实例组成的数组，如果数组中存在非 `Promise` 的示例，则 `allPromise` 的状态直接为 `reject`。  

	`allPromise` 的状态由 `p1/p2/p3` 共同决定，三个全部 `resolve` 则 `allPromise` 转变为 `resolve` ，其中任意一个出现 `reject` ，则 `allPromise` 转变为 `reject` 。  

* `Promise.race()`
	
	`Promise.race` 方法同样用于将多个Promise实例，包装成一个新的Promise实例。  

	```js
	var allPromise = Promise.all([p1, p2, p3])
	```

	与 `Promise.all` 不同的是，如果 `p1/p2/p3` 中有任意一个状态先发生了变化，则 `allPromise` 的状态也会跟着转变，并且状态与最先发生状态改变的 `promise` 一致。  

###### d. 实际应用

* 图片加载

	```js
	var preloadImg = function(url){
		return new Promise(function(resolve, reject){
			var img = new Image();
			img.onload = resolve;
			img.onerror = reject;
			img.src = url;
		})
	}
	
	// 调用
	var img1 = preloadImg('./img/test1.png');
	var img2 = preloadImg('./img/test2.png');
	var img3 = preloadImg('./img/test3.png');
	var img4 = preloadImg('./img/test4.png');
	Promise
		.all([img1, img2, img3, img4])
		.then(function(){
			// all img loaded
			$('.loading').hide();
		})
		.catch(function(err){
			// catch err
			console.log(err);
		})
	```

* `Promise` 风格的文件读写

	```js
	var fs = require('fs');
	var readFile = function(path){
		return new Promise(function(resolve, reject){
			fs.readFile(path, 'utf8', function(err, data) {
				if(err) {
					reject(err);
				} else {
					resolve(data);
				}
			});
		})
	}
	var writeFile = function(path, data){
		return new Promise(function(resolve, reject){
			fs.writeFile(path, data, 'utf-8', function(err, data){
				if(err){
					reject(err);
				} else {
					resolve(data);
				}
			})
		})
	}

	// 调用
	readFile('./test.json')
		.then(function(data){
			console.log(data);
			return data;
		})
		.then(function(data){
			// replace all 'abc' to 'ABC'
			writeFile('./test.json', data.replace(/abc/g, 'ABC'));
		})
		.catch(function(err){
			console.log(err);
		})
	```

#### 2、Generator

[Generator - MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Generator)

想象这样的一个场景：  

> 当你执行一个函数的时候，需要在某个时间点停下来等待另一个操作完成，并且拿到这个操作的执行结果，然后继续执行。

这样的场景就是 `ES6` 的生成器需要解决的问题。  

###### a. 基本理解

* 生成器本质上是一种特殊的[迭代器](https://zh.wikipedia.org/wiki/%E8%BF%AD%E4%BB%A3)，迭代最简单的例子如下：  

	```js
	for(var i = 0; i < 10; i++){
		// 每一次循环就是一次迭代，每次迭代都依赖上一次的 i 的值
		console.log(i);
	}
	```

	而生成器作为一种特殊的迭代器就是它的每一次迭代都是可控的，详情下面将具体描述。  

* 生成器形式上是一种函数，只不过比普通的函数 `function` 多一个 `*` ，即 `function*(){}`。  

###### b. 基本API

* `function*(){}`
* `yield`
* `yield*`
* `Generator.prototype.next()`
* `Generator.prototype.return()`
* `Generator.prototype.throw()`

###### c. 详解

* 声明 `Generator` 函数

	```js
	var test = function*(i){
		console.log(i);
		yield i++;
		console.log(i);
		var j = yield i++;
		console.log(j)
	}
	```

	上面的例子就是一个简单的 `Generator` 函数，可以发现，函数声明是多个一个 `*`，并且函数体内部出现了 `yield` 语句。现在让我们来执行这个 `Generator` 函数。  

	```js
	var g = test(1);  // no console result
	g.next();  // 1 {value: 1, done: false}
	var ret = g.next();  // 2
	g.next(ret.value + 1);  // 3 {value: undefined, done: true}
	```

	从这个简单的小例子，我们不难看出：  
	1. 生成器函数在定义并调用时，并不会即时执行。  
	2. 每次调用生成器函数的 `next` 方法，会执行生成器函数中的语句直到遇到 `yield` 语句，然后生成器函数会拿到 `yield` 语句后的返回值，跳出当前函数执行，等待下一次 `next` 调用。  
	3. `yield` 语句会把当前的返回值放到一个对象的 `value` 中，整体返回，以备调用。  
	4. 每次调用 `next` 语句，都会返回一个对象，包括两个字段，`value` 字段就是 `yield` 语句的返回值，`done` 字段标识生成器函数是否执行完毕。  

	明白了这些，让我们在回头看上面的代码，捋一下它的执行顺序。

	```js
	var test = function*(i){
		console.log(i);
		yield i++;
		console.log(i);
		var j = yield i++;
		console.log(j)
	}

	var g = test(1);
	g.next();
	var ret = g.next();
	g.next(ret.value + 1);
	```












