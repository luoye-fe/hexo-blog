title: js中的异步流程控制--Promise/Generators/Async/Await
date: 2016-06-09 08:05:14
tags:
---

异步I/O、事件驱动使JS这个单线程语言在不阻塞的情况下可以并行的执行很多任务，这带来了性能的极大提升，并且更加符合人们的自然认识（烧一壶水，期间你肯定不会等着水烧开再去做别的事，异步才是正常的啊！）。然而异步风格也给流程控制，错误处理带来了更多的麻烦。


## 异步流程控制的解决方案

#### （一）回调

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

#### （二）事件监听（观察者模式）

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

#### （三）异步流程控制库

为了优雅的解决异步流程控制的问题，伟大的猿们前赴后继，产出了很多方案，造就了不少优秀的库，包括但不限于 `q` `co` `async` 等。  

这些库的具体实现或使用方式不在本文的谈论范围，暂时跳过。  

#### （四）新标准、新未来

> 重点来了！

现在已经是2016年了，`ES` 的标准一代快过一代，有了 `bable` 这样的工具，甚至 `ES7` 都不再是不可触及的 `feture`了，新的标准当然对异步控制做出了很多努力，让我们一个一个来看。  

* Promise

	[Promise－MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise)

	如上文中提到的示例，如果一个函数的回调层级过深，简直是掉入地狱的体验，那么如果能够把回调的层级抹平，会不会好很多呢？`Promise` 正是一个完美的解决方案。  

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



