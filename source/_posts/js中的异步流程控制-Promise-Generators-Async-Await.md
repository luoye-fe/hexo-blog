title: js中的异步流程控制--Promise/Generator/Async/Await
date: 2016-06-09 08:05:14
tags:
---

> 长文预警 ～

异步I/O、事件驱动使JS这个单线程语言在不阻塞的情况下可以并行的执行很多任务，这带来了性能的极大提升，并且更加符合人们的自然认识（烧一壶水，期间你肯定不会等着水烧开再去做别的事，异步才是正常的啊！）。然而异步风格也给流程控制，错误处理带来了更多的麻烦。

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

* `Promise.resolve()` 
* `Promise.reject()`
* `Promise.prototype.then()`
* `Promise.prototype.catch()`
* `Promise.all()`
* `Promise.race()`

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
* `Generator.prototype.next()`
* `Generator.prototype.return()`
* `Generator.prototype.throw()`
* `yield*`

###### c. 详解

* `Generator` 函数

	```js
	var test = function*(){
		yield 'hello';
		yield 'world';
		yield '!';
		return 'func end';
	}
	var helloWorld = test();
	```

	上面的例子就是一个简单的 `Generator` 函数，可以发现，函数声明是多个一个 `*`，并且函数体内出现了多个 `yield` 语句和 `return` 语句，即该生成器函数存在四种迭代状态： `hello` `world` `!` `return`

	但是当我们执行上述代码的时候，发现并没有即时的执行，返回的也不是它的执行结果，而是一个生成器对象，只有当调用这个生成器对象的 `next` 方法，才会依次的执行函数语句，直到遇到 `yield` 语句或 `return` 语句。  

	```js
	helloWorld.next(); // {value: "hello", done: false}
	helloWorld.next(); // {value: "world", done: false}
	helloWorld.next(); // {value: "!", done: false}
	helloWorld.next(); // {value: "func end", done: true}
	helloWorld.next(); // {value: undefined, done: true}
	```

	<img src="./1.png" style="margin-left: 0;">
	
	让我们梳理一下上述代码的执行流程。  

	第一次调用 `next`： 生成器函数开始执行，遇到 `yield` 语句，暂停执行。`next` 返回一个对象，其中将当前 `yeild` 语句的值 `hello` 作为返回对象的 `value` 字段。`done` 字段为 `false`，迭代未结束。

	第二次调用 `next`： 从上一个 `yield` 语句开始执行，遇到 `yield` 语句，暂停执行。`next` 返回一个对象，其中将当前 `yeild` 语句的值 `world` 作为返回对象的 `value` 字段。`done` 字段为 `false`，迭代未结束。

	第三次调用 `next`： 从上一个 `yield` 语句开始执行，遇到 `yield` 语句，暂停执行。`next` 返回一个对象，其中将当前 `yeild` 语句的值 `!` 作为返回对象的 `value` 字段。`done` 字段为 `false`，迭代未结束。

	第四次调用 `next`： 从上一个 `yield` 语句开始执行，遇到 `return` 语句，结束执行。`next` 返回一个对象，其中将当前 `return` 语句的值 `func end` 作为返回对象的 `value` 字段。`done` 字段为 `true`，迭代结束。

	第五次调用 `next`： 生成器函数已经迭代（运行）完毕，`next` 方法始终返回 `{value: undefined, done: true}`

	让我们再用一个例子来了解一下 `yield` 语句的执行流程：  

	```js
	var gen = function*(){
		console.log('func start');
		var i = 0;
		while(i < 6){
			console.log('yield start');
			yield i;
			console.log('yield end'); 
			i++;
		}
		console.log('func end');
	}
	var genEx = gen();
	```

	<img src="./2.png" style="margin-left: 0;">

	首次调用 `next` ，函数开始执行，遇到 `yield` 暂停执行，将 `yield` 语句后的表达式运行后返回，当作 `next` 方法返回值的 `value` 字段，依次调用 `next` ，从上次 `yield` 处继续运行，直到遇到下一个 `yield`，循环往复。  

* `yield` 语句
	
	通过上面的例子， `yield` 语句的特性已经很明显：  

	* `yield` 语句会暂停生成器函数的执行

	* `yield` 语句后表达式的运行结果将作为 `next` 语句返回值中的 `value` 字段

* `Generator.prototype.next()`

	`next` 语句的返回值有两个字段 `value` 和 `done` ，`value` 为当前 `next` 指向的 `yield` 语句的返回值，`done` 标识当前生成器函数是否迭代完毕。  

	`next` 方法还可以接受任意一个参数，该参数将作为上一个 `yield` 返回值。 

	```js
	var gen = function*(i){
		var i = 0;
		while(true){
			var reset = yield i;
			if(reset){
				i = reset;
			}
			i++;
		}
	}
	var genEx = gen();

	genEx.next();  // {value: 1, done: false}
	genEx.next();  // {value: 2, done: false}
	genEx.next(10);  // {value: 11, done: false}
	```

	上面的代码实现了一个无限的迭代器，在每次运行到 `yield` 语句时，如果调用指向此次 `yield` 语句的 `next` 方法没有参数，那么 `reset` 的值始终是 `undefined`。只有在调用 `next` 方法传入了参数，此次执行 `yield` 语句时，`yield` 语句的返回值将变为 `next` 传入的参数。这样的特性能够让我们用同步的方式写出异步执行的代码，具体例子下文。  

* `Generator.prototype.return()`

	当我们想在外部结束生成器函数的迭代，可以使用 `return` 方法，并将 `return` 方法的参数作为返回值。  

	```js
	var gen = function*(){
		yield 1;
		yield 2;
		yield 3;
	}
	var genEx = gen();

	genEx.next();  // {value: 1, done: false}
	genEx.return('end');  // {value: 'end', done: true}
	genEx.next();  // {value: undefined, done: true}
	```

* `Generator.prototype.return()`

	`throw` 方法允许我们在生成器函数外部抛出错误，并在内部捕获。  

	```js
	var gen = function*(){
		try{
			yield;
		}catch(e){
			console.log('inner error: ' + e);
		}
	}
	var genEx = gen();
	genEx.next();

	try{
		genEx.throw('a');
		genEx.throw('b');
	}catch(e){
		console.log('outer error: ' + e);
	}

	// inner error: a
	// outer error: b
	```

	第一次抛出错误，被生成器函数捕获到，第二次再抛出，由于 `catch` 语句已经在第一次执行过了，所以内部无法再次捕获错误，从而在外部的 `try catch` 语句中可以捕获到错误。  

* `yield*`
	
	如果想在生成器函数中调用另一个生成器函数，将会用到 `yield*` 语句。  

	```js
	var gen1 = function*(){
		yield '1';
		yield '2';
	}
	var gen2 = function*(){
		yield 'a';
		yield* gen1();
		yield 'b';
	}
	var genEx = gen2();

	genEx.next(); // {value: "a", done: false}
	genEx.next(); // {value: "1", done: false}
	genEx.next(); // {value: "2", done: false}
	genEx.next(); // {value: "b", done: false}
	genEx.next(); // {value: undefined, done: true}
	```

###### d. 实际应用

* 异步 `Ajax` 请求

	```js
	var gen = function*(url){
		// fetch: 原生的ajax请求API
		var result = yield fetch(url);
		console.log(result);
	}

	var genEx = gen('https://api.github.com/users/github');
	var result = genEx.next();
	
	result.value.then(function(res){
		console.log(res);
		return res.json();
	}).then(function(data){
		genEx.next(data.bio);  // How people build software.
	})
	```

	上面的代码中，第一次调用 `next` 方法，开始请求，拿到返回结果后，用结果中的 `value`（ `fetch` 返回的是一个 `Promise`，所以需要 `then` 方法调用），调用下一次 `then` 从而执行生成器函数中 `yield` 后面的代码。  

	可以看出，虽然生成器函数将异步操作表示的很简洁，但是流程管理并不是很直接，即何时执行第一阶段，何时执行第二阶段并不能很好的向使用者展示。  

#### 3、Async/Await

从回调，到 `Promise`，再到 `Generator` 函数，js的异步流程控制一直在进化，但是每种解决方法都无形的增加了额外的复杂度，都需要理解底层的运行机制才能很好的运用。  

而 `ES7` 提出的 `Async/Await`，大概也许可能是 JavaScript 中最好的异步解决方案。  

###### a. 实例
	
* 异步读取文件

	```js
	var fs = require('fs');
	// 与上文一致
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

	// 调用
	var asyncReadFile = async function(){
		var file1 = await readFile('./test1.json');
		var file2 = await readFile('./test2.json');
		console.log(file1);
		console.log(file2);	
	}
	asyncReadFile();
	```

	如果把上面的代码写成 `Geneerator` 风格，你会发现两者很相似。  

	```js
	var asyncReadFile = function*(){
		var file1 = yield readFile('./test1.json');
		var file2 = yield readFile('./test2.json');
		console.log(file1);
		console.log(file2);	
	}
	```

	对比之后，其实 `async` 函数就是把 `*` 替换成 `async`，把 `yield` 替换成 `await`。  

	可以说，`async` 其实就是对 `Geneerator` 的语法糖，只不过多包了一层，改进了很多。  

	第一，使用 `async` 函数不用再手动的调用 `next` 方法来执行每一次迭代

	第二，更好的语义，`async` 表示这个函数是一个异步函数，`await` 表示此后的操作需要等待此步操作完成

	第三，侵入性更低，原生的 `try catch` 语句能处理错误，`async` 函数中的 `await` 语句不用做特殊处理，`Promise` 可以，原始的同步操作也可以  

	第四，更直观、更灵活的调用，`async` 函数返回的是一个 `Promise` 对象，异步操作完成后可以直接用 `then` 方法进行下一步操作  

	第五，简单的API，只有 `async` 和 `await` 两个API，`async` 用来声明一个异步函数，`await` 用来等待一个异步操作  
	
* `sleep` 函数

	上文我们用 `Promise` 实现了一个异步风格的 `sleep` 函数，现在让我们看看如何用同步的风格实现并使用它。  

	```js
	var sleep = function(ms){
		return new Promise(function(resolve, reject){
			setTimeout(resolve, ms)
		})
	}
	
	var sleepEx = async function(){
		console.log('begin');
		await sleep(1000);
		console.log('end after 1000ms');
	}
	sleepEx();
	```

	完美～

###### b. 如何使用

`async` `await` 特性属于ES7的新特性，目前的ES运行环境中并没有实现这样的功能，但是借助 `babel`，我们可以很方便的使用这些新特性。  

这个展开讲又是一个大话题～贴一个 `bable` 转换代码的网址：[Babel transform online](https://babeljs.io/repl/#?evaluate=true&lineWrap=false&presets=es2015%2Cstage-0)  

如何在线下使用，自行谷歌，或者，再来一篇？哈哈  

### 五、结束

长长的文章终于结束了，呼～  

主要的目的就是对异步流程的解决方案进行一下梳理，加深对js异步特性的理解。最推荐的方式还是ES7的新特性，毕竟是既有的新标准，使用的过程还能学习下 `babel` 的配置，哈哈。

