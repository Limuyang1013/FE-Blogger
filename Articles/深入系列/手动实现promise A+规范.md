#### 概念

所谓`Promise`，简单说就是一个容器，里面保存着某个未来才会结束的事件（通常是一个异步操作）的结果。从语法上说，`Promise` 是一个对象，从它可以获取异步操作的消息。`Promise` 提供统一的 API，各种异步操作都可以用同样的方法进行处理。

#### 简易版Promise的实现

在实现一个能通过`Promise A+`规范测试的`Promise`版本之前，我们先写一个简易版的`Promise`，结合我们平常使用`Promise`的姿势：

```javascript
const promise = new Promise(function(resolve, reject) {
  // ... some code

  if (/* 异步操作成功 */){
    resolve(value);
  } else {
    reject(error);
  }
});

```
对照[Promise A+规范](https://promisesaplus.com/#notes)开始撸代码。

```
1.1 “promise” is an object or function with a then method whose behavior conforms to this specification.
1.2 “thenable” is an object or function that defines a then method.
1.3 “value” is any legal JavaScript value (including undefined, a thenable, or a promise).
1.4 “exception” is a value that is thrown using the throw statement.
1.5 “reason” is a value that indicates why a promise was rejected.
```
先实现一个`Promise`的构造函数：

```javascript
function Promise(executor) {
	let self = this
	self.status = 'pending' // 一个promise必须处于三种状态之一： 请求态（pending）， 完成态（fulfilled），拒绝态（rejected）
	self.value = null
	self.onResolveCallbacks = [] // Promise resolve的时候可能会有多个回调 
	self.onRejectCallbacks = [] // 同上
	executor(resolve, reject) // 执行executor
}
```
- 在构造函数内部先使用一个常量`self`，当异步执行的时候可以获取正确的`this`对象
- `Promise`的初始状态应该是`pending`
- `value`变量用于存储`resolve`或者`reject`的值，这里其实同时表示了规范中的`value/reason`
- 当执行完`Promise`之后当前`Promise`的`status`可能还是`pending`，这个时候要把`then`方法中传入的回调存储到对应的`callBackList`

我们知道`Promise`接受一个函数作为参数，这个函数有`resolve`和`reject`两个函数作为参数，`resolve`函数可以将`Promise`的状态从`pending`转为`fulfilled`，并将异步操作的结果作为参数传递出去，`reject`函数可以将`Promise`的状态从`pending`转为`rejected`，并将异步操作失败报出的错误作为参数传递出去，下面我们来实现这两个函数：

```javascript
function Promise(executor) {
	let self = this
	self.status = 'pending'
	self.value = null
	self.onResolveCallbacks = []
	self.onRejectCallbacks = []

	function resolve(value) {
		if (self.status === 'pending') {
			self.status = 'fulfilled'
			self.value = value
			self.onResolveCallbacks.forEach(cb => cb(self.value))
		}
	}

	function reject(reason) {
		if (self.status === 'pending') {
			self.status = 'rejected'
			self.value = reason
			self.onRejectCallbacks.forEach(cb => cb(self.value))
		}
	}

	try {
		executor(resolve, reject)
	} catch (e) {
		reject(e)
	}
}
```
这里有一点需要解释的是，执行`executor`有可能会出错就像下面这样，如果出错了`Promise`应该`reject`这个出错的值：

```javascript
new Promise(function(resolve, reject) {
  throw 2
})

```
下一步，实现逻辑较为复杂的`then`函数，也是整个`Promise`的精髓所在：

```
A promise must provide a then method to access its current or eventual value or reason.

A promise’s then method accepts two arguments:

promise.then(onFulfilled, onRejected)
```
因为`then`方法可以链式调用，所以我们把它构造在`Promise`的原型链上：

```javascript
Promise.prototype.then = function(onFulfilled, onRejected) {
	// 内部实现
}
```
根据标准`2.2.1`和`2.2.7`我们可以写出如下代码：

```javascript
Promise.prototype.then = function(onFulfilled, onRejected) {
	let self = this
	onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : v => v // 2.2.1.1
	onRejected = typeof onRejected === 'function' ? onRejected : r => {
		throw r
	} // 2.2.1.2
	if (self.status === 'pending') {
		self.onResolveCallbacks.push(onFulfilled)
		self.onRejectCallbacks.push(onRejected)
	} else if (self.status === 'fulfilled') {
		onFulfilled(self.value)
	} else if (self.status === 'rejected') {
		onRejected(self.value)
	}
}
```
为了透传我们在`onFulfilled/onRejected`不是一个函数的时候也创造了一个函数返回当前`onFulfilled/onRejected`的值，当`status`处于`pending`状态，我们不能确定调用`onFulfilled`还是`onRejected`，只有等状态确定后才可以处理，所以这里把两个`callback`存入回调数组里面，如果是确定状态则调用对应的方法即可，这就是最简易版本的`Promise`，来试一下：

```javascript
		new Promise(function(resolve, reject) {
			setTimeout(() => {
				resolve(1)
			})
		}).then((value) => {
			console.log(value) // 打印结果 1
		})
		
```

#### 实现一个符合Promise/A+规范的Promise

以上面的简易版为基础，继续改造`then`方法，规范中有提到说`then`需要返回一个`Promise`，这也是它可以链式操作的关键所在：

```
then must return a promise [3.3].

promise2 = promise1.then(onFulfilled, onRejected);
```
虽然说规范里面允许我们返回同一个`Promise`，但是我们这里遵循大多数`Promise`的实现给它返回一个新的`Promise`：

```javascript
Promise.prototype.then = function(onFulfilled, onRejected) {
	let self = this
	onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : v => v // 2.2.1.1
	onRejected = typeof onRejected === 'function' ? onRejected : r => {
		throw r
	} // 2.2.1.2
	let returnedPromise = null // 2.2.7
	if (self.status === 'pending') {
		return returnedPromise = new Promise((resolve, reject) => {
			self.onResolveCallbacks.push(onFulfilled)
			self.onRejectCallbacks.push(onRejected)
		})
	} else if (self.status === 'fulfilled') {
		return returnedPromise = new Promise((resolve, reject) => {
			onFulfilled(self.value)
		})
	} else if (self.status === 'rejected') {
		return returnedPromise = new Promise((resolve, reject) => {
			onRejected(self.value)
		})
	}
}
```
根据规范的定义，我们这里还需要定义一个`[[Resolve]](promise2, x) `解析函数，解析`onFulfilled`或`onRejected`的返回值，同时对两个方法执行期间抛出的错误进行`reject`：

```javascript
Promise.prototype.then = function(onFulfilled, onRejected) {
	let self = this
	onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : v => v // 2.2.1.1
	onRejected = typeof onRejected === 'function' ? onRejected : r => {
		throw r
	} // 2.2.1.2
	let returnedPromise = null // 2.2.7
	if (self.status === 'pending') {
		return returnedPromise = new Promise((resolve, reject) => {
			self.onResolveCallbacks.push(() => {
				try {
					let x = onFulfilled(self.value)
					ResolutionProcedure(returnedPromise, x, resolve, reject)
				} catch (e) {
					reject(e)
				}
			})
			self.onRejectCallbacks.push(() => {
				try {
					let x = onRejected(self.value)
					ResolutionProcedure(returnedPromise, x, resolve, reject)
				} catch (e) {
					reject(e)
				}
			})
		})
	} else if (self.status === 'fulfilled') {
		return returnedPromise = new Promise((resolve, reject) => {
			try {
				let x = onFulfilled(self.value)
				ResolutionProcedure(returnedPromise, x, resolve, reject)
			} catch (e) {
				reject(e)
			}
		})
	} else if (self.status === 'rejected') {
		return returnedPromise = new Promise((resolve, reject) => {
			try {
				let x = onRejected(self.value)
				ResolutionProcedure(returnedPromise, x, resolve, reject)
			} catch (e) {
				reject(e)
			}
		})
	}
}

function ResolutionProcedure(promise, x, resolvePromise, rejectPromise) {
	// 内部实现
}
```
接下来只需要关注这个`ResolutionProcedure`函数的内部实现，这里其实规范都给出了所有的详细步骤，按照规范来做就行：

```javascript
function ResolutionProcedure(promise, x, resolvePromise, rejectPromise) {
	try {
		if (promise === x) {
			return rejectPromise(new TypeError('2.3.1'))
		}
		if (x instanceof Promise) //2.3.2
			x.then(((value)=>{
				ResolutionProcedure(promise, value, reslove, reject);
			},(reason)=>{
				reject(reason)
			}));
		let called = false // 2.3.3.3.3
		if (x !== null && (typeof x === 'object' || typeof x === 'function')) { // 2.3.3
			try { // 2.3.3.1
				let then = x.then
				if (typeof then === 'function') { // 2.3.3.3
					then.call(x, (value) => {
						if (called) return
						called = true
						return ResolutionProcedure(promise, value, resolvePromise, rejectPromise)
					}, (reason) => {
						if (called) return
						called = true
						return rejectPromise(reason)
					})
				} else {
					return resolvePromise(x) // 2.3.4
				}
			} catch (e) {
				if (called) return
				called = true
				return rejectPromise(e) // 2.3.3.2
			}
		} else {
			return resolvePromise(x) // 2.3.3.4
		}
	} catch (e) {
		return rejectPromise(e)
	}
}

```
然后我们注意到规范`2.2.4`中说了只有在执行栈包含平台代码的时候才可以调用`onFulfilled`和`onRejected`：

```
onFulfilled or onRejected must not be called until the execution context stack contains only platform code.
```
解释文案中也说了其实就是异步执行，我们可以用`setTimeout`或`setImmediate`来做到这一点：

```javascript
Promise.prototype.then = function(onFulfilled, onRejected) {
	let self = this
	onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : v => v // 2.2.1.1
	onRejected = typeof onRejected === 'function' ? onRejected : r => {
		throw r
	} // 2.2.1.2
	let returnedPromise = null // 2.2.7
	if (self.status === 'pending') {
		return returnedPromise = new Promise((resolve, reject) => {
			self.onResolveCallbacks.push(() => {
				setTimeout(() => {
					try {
						let x = onFulfilled(self.value)
						ResolutionProcedure(returnedPromise, x, resolve, reject)
					} catch (e) {
						reject(e)
					}
				}, 0)
			})
			self.onRejectCallbacks.push(() => {
				setTimeout(() => {
					try {
						let x = onRejected(self.value)
						ResolutionProcedure(returnedPromise, x, resolve, reject)
					} catch (e) {
						reject(e)
					}
				}, 0)
			})
		})
	} else if (self.status === 'fulfilled') {
		return returnedPromise = new Promise((resolve, reject) => {
			setTimeout(() => {
				try {
					let x = onFulfilled(self.value)
					ResolutionProcedure(returnedPromise, x, resolve, reject)
				} catch (e) {
					reject(e)
				}
			}, 0)
		})
	} else if (self.status === 'rejected') {
		return returnedPromise = new Promise((resolve, reject) => {
			setTimeout(() => {
				try {
					let x = onRejected(self.value)
					ResolutionProcedure(returnedPromise, x, resolve, reject)
				} catch (e) {
					reject(e)
				}
			}, 0)
		})
	}
}
```

#### 测试

这里用官方提供的`promises-aplus-tests `来验证我们写出来的库是否符合规范，为了跑测试我们还需要加上`deferred`方法：

```
Promise.deferred = function() {
	let defer = {}
	defer.promise = new Promise((resolve, reject) => {
		defer.resolve = resolve
		defer.reject = reject
	})
	return defer
}

module.exports = Promise
```
跑一下测试：

```
npm install -g promises-aplus-tests 

promises-aplus-tests ./promise.js
```
<img width="650" alt="WeChat747a5c781800c28642a62c8ad2150198" src="https://user-images.githubusercontent.com/11991572/56417937-0ba73700-62c8-11e9-91d9-6c88e3fefcb9.png">

说明我们写的库是符合的`Promise/A+`标准的

#### 扩展

`Promise/A+`规范只是定义了`then`方法，但是`Promise`本身还有一些其他的方法，我们也可以实现一下：

- catch

```javascript
Promise.prototype.catch = function(onRejected) {
	return this.then(null, onRejected)
}
```
- race

```javascript
Promise.prototype.race = function(values) {
	return new Promise(function(resolve, reject) {
		for (let i = 0, len = values.length; i < len; i++) {
			values[i].then(resolve, reject)
		}
	})
}
```
- all

```javascript
Promise.all = function(promises) {
	return new Promise(function(resolve, reject) {
		let resolvedCounter = 0
		let promiseNum = promises.length
		let resolvedValues = new Array(promiseNum)
		for (let i = 0; i < promiseNum; i++) {
			(function(i) {
				Promise.resolve(promises[i]).then(function(value) {
					resolvedCounter++
					resolvedValues[i] = value
					if (resolvedCounter === promiseNum) {
						return resolve(resolvedValues)
					}
				}, function(reason) {
					return reject(reason)
				})
			})(i)
		}
	})
}
```
- finally

```javascript
Promise.prototype.finally = function(callback) {
	let constructor = this.constructor
	return this.then(
		function(value) {
			return constructor.resolve(callback()).then(function() {
				return value
			})
		},
		function(reason) {
			return constructor.resolve(callback()).then(function() {
				return constructor.reject(reason)
			})
		}
	)
}
```

最后奉上实现后的全部代码：

```javascript
function Promise(executor) {
	let self = this
	self.status = 'pending'
	self.value = null
	self.onResolveCallbacks = []
	self.onRejectCallbacks = []

	function resolve(value) {
		if (self.status === 'pending') {
			self.status = 'fulfilled'
			self.value = value
			self.onResolveCallbacks.forEach(cb => cb(self.value))
		}
	}

	function reject(reason) {
		if (self.status === 'pending') {
			self.status = 'rejected'
			self.value = reason
			self.onRejectCallbacks.forEach(cb => cb(self.value))
		}
	}

	try {
		executor(resolve, reject)
	} catch (e) {
		reject(e)
	}
}

Promise.prototype.then = function(onFulfilled, onRejected) {
	let self = this
	onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : v => v // 2.2.1.1
	onRejected = typeof onRejected === 'function' ? onRejected : r => {
		throw r
	} // 2.2.1.2
	let returnedPromise = null // 2.2.7
	if (self.status === 'pending') {
		return returnedPromise = new Promise((resolve, reject) => {
			self.onResolveCallbacks.push(() => {
				setTimeout(() => {
					try {
						let x = onFulfilled(self.value)
						ResolutionProcedure(returnedPromise, x, resolve, reject)
					} catch (e) {
						reject(e)
					}
				}, 0)
			})
			self.onRejectCallbacks.push(() => {
				setTimeout(() => {
					try {
						let x = onRejected(self.value)
						ResolutionProcedure(returnedPromise, x, resolve, reject)
					} catch (e) {
						reject(e)
					}
				}, 0)
			})
		})
	} else if (self.status === 'fulfilled') {
		return returnedPromise = new Promise((resolve, reject) => {
			setTimeout(() => {
				try {
					let x = onFulfilled(self.value)
					ResolutionProcedure(returnedPromise, x, resolve, reject)
				} catch (e) {
					reject(e)
				}
			}, 0)
		})
	} else if (self.status === 'rejected') {
		return returnedPromise = new Promise((resolve, reject) => {
			setTimeout(() => {
				try {
					let x = onRejected(self.value)
					ResolutionProcedure(returnedPromise, x, resolve, reject)
				} catch (e) {
					reject(e)
				}
			}, 0)
		})
	}
}

Promise.prototype.catch = function(onRejected) {
	return this.then(null, onRejected)
}

Promise.prototype.race = function(values) {
	return new Promise(function(resolve, reject) {
		for (let i = 0, len = values.length; i < len; i++) {
			values[i].then(resolve, reject)
		}
	})
}

Promise.all = function(promises) {
	return new Promise(function(resolve, reject) {
		let resolvedCounter = 0
		let promiseNum = promises.length
		let resolvedValues = new Array(promiseNum)
		for (let i = 0; i < promiseNum; i++) {
			(function(i) {
				Promise.resolve(promises[i]).then(function(value) {
					resolvedCounter++
					resolvedValues[i] = value
					if (resolvedCounter === promiseNum) {
						return resolve(resolvedValues)
					}
				}, function(reason) {
					return reject(reason)
				})
			})(i)
		}
	})
}

Promise.prototype.finally = function(callback) {
	let constructor = this.constructor
	return this.then(
		function(value) {
			return constructor.resolve(callback()).then(function() {
				return value
			})
		},
		function(reason) {
			return constructor.resolve(callback()).then(function() {
				return constructor.reject(reason)
			})
		}
	)
}

function ResolutionProcedure(promise, x, resolvePromise, rejectPromise) {
	try {
		if (promise === x) {
			return rejectPromise(new TypeError('2.3.1'))
		}
		if (x instanceof Promise) //2.3.2
			x.then(((value) => {
				ResolutionProcedure(promise, value, reslove, reject)
			}, (reason) => {
				reject(reason)
			}))
		let called = false // 2.3.3.3.3
		if (x !== null && (typeof x === 'object' || typeof x === 'function')) { // 2.3.3
			try { // 2.3.3.1
				let then = x.then
				if (typeof then === 'function') { // 2.3.3.3
					then.call(x, (value) => {
						if (called) return
						called = true
						return ResolutionProcedure(promise, value, resolvePromise, rejectPromise)
					}, (reason) => {
						if (called) return
						called = true
						return rejectPromise(reason)
					})
				} else {
					return resolvePromise(x) // 2.3.4
				}
			} catch (e) {
				if (called) return
				called = true
				return rejectPromise(e) // 2.3.3.2
			}
		} else {
			return resolvePromise(x) // 2.3.3.4
		}
	} catch (e) {
		return rejectPromise(e)
	}
}

Promise.deferred = function() {
	let defer = {}
	defer.promise = new Promise((resolve, reject) => {
		defer.resolve = resolve
		defer.reject = reject
	})
	return defer
}

module.exports = Promise

```
