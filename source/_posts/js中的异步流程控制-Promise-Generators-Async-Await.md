title: js中的异步流程控制--Promise/Generators/Async/Await
date: 2016-06-09 08:05:14
tags:
---

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

所谓的 `Promise` ，就是一个特殊的用于传递异步信息的对象，它代表一个未完成的但是将会完成的操作。也就是说，`Promise` 代表了某个未来才会知道结果的事件（通常是一个异步操作），并且为这个异步事件提供统一的 `API`，能够让使用者精确的控制每一个流程。  

说的很复杂，总之一句话，用 `Promise` 能够很好的让我们控制异步操作。继续往下看。  

###### a. 基本 API

* Promise.resolve() 
* Promise.reject()
* Promise.prototype.then()
* Promise.prototype.catch()
* Promise.all()
* Promise.race()

###### b. 基本理解

* 一个 `Promise` 对象，存在三种状态， `pending(进行中)`、`resolve(已完成)`、`reject(已失败)`。一个异步操作的开始，对应着 `Promise` 的 `pending` 状态，异步操作的结束，对应着另两种状态，当异步操作成功时，对应着 `resolve`状态，失败时对应着 `reject`状态。

* `Promise` 的状态如果发生改变，就不能再被更改，并且，只能由 `pending` 向另外两种状态转变，不能逆，也不能 `resolve` 和 `reject` 互相转化。  

###### c. 实践

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

	`Promise` 实例创建后，可以用 `then` 方法，继续异步操作成功或失败之后的操作。  

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

	在 `then` 方法中，第二个参数可以对 `Promise` 中的错误进行处理，但是如果一个链式调用的 `Promise` 中，对每一个 `then` 方法进行错误处理，无疑是很挫的方式，`Promise` 为我们提供了一个更加方便的错误处理方式。  

	
	
	







	在 `Promise` 中，`then` 方法始终返回一个新的 `Promise` 对象，这样，我们就可以实现 ** 链式 ** 的调用，如：  

	```js
	sleep(1000)
		.then(function(){
			console.log('1000s gone')
		})
		.then(function(){
			console.log('will exec immediately after 1000ms')
		})
	```

	很显然，这样的调用方式可以


```js
var f1 = function(){
	return new Promise(function(resolve, reject) {
		setTimeout(function() {
			console.log(Date.now());
			resolve();
		}, 2000)
	})
}

var f2 = function(){
	return new Promise(function(resolve, reject) {
		setTimeout(function() {
			console.log(Date.now());
			resolve();
		}, 2000)
	})
}

var f3 = function() {
	console.log(Date.now());
}
f1().then(function(){
	f2();
}).then(function(){
	f3();
})
```



