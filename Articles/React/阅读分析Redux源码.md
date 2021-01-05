之前在`Medium`上看过一篇很有意思的文章： [The deepest reason why modern JavaScript frameworks exist](https://medium.com/dailyjs/the-deepest-reason-why-modern-javascript-frameworks-exist-933b86ebc445) 文章里面大概描述了这么一个事实：我们用着一系列现代开发框架(Vue、React、Angular)，但是我们却忽略了他们之所以存在的最重要的意义：

> 管理不断变化的 state 非常困难。如果一个 model 的变化会引起另一个 model 变化，那么当 view 变化时，就可能引起对应 model 以及另一个 model 的变化，依次地，可能会引起另一个 view 的变化。直至你搞不清楚到底发生了什么。state 在什么时候，由于什么原因，如何变化已然不受控制。 当系统变得错综复杂的时候，想重现问题或者添加新功能就会变得举步维艰。

![why modern javascript frameworks exist](https://user-images.githubusercontent.com/11991572/59352947-7cad1d00-8d54-11e9-965d-c8095161d8aa.png)

无论是`Vuex`还是`Redux`，其实都是为了解决我们`state`的管理的一系列问题而产生的，这里主要分析`Redux`的原理。

先看一眼`Redux`源码的文件结构：
```
├── applyMiddleware.js
├── bindActionCreators.js
├── combineReducers.js
├── compose.js
├── createStore.js
├── index.js
└── utils
    ├── actionTypes.js
    ├── isPlainObject.js
    └── warning.js

```
基本都包含了我们常用的一些`API`外加一些工具类方法，那就先从入口文件开始看起。

#### index

```javascript
import createStore from './createStore'
import combineReducers from './combineReducers'
import bindActionCreators from './bindActionCreators'
import applyMiddleware from './applyMiddleware'
import compose from './compose'
import warning from './utils/warning'
import __DO_NOT_USE__ActionTypes from './utils/actionTypes'

/*
 * This is a dummy function to check if the function name has been altered by minification.
 * If the function has been minified and NODE_ENV !== 'production', warn the user.
 */
function isCrushed() {}

if (
  process.env.NODE_ENV !== 'production' &&
  typeof isCrushed.name === 'string' &&
  isCrushed.name !== 'isCrushed'
) {
  warning(
    'You are currently using minified code outside of NODE_ENV === "production". ' +
      'This means that you are running a slower development build of Redux. ' +
      'You can use loose-envify (https://github.com/zertosh/loose-envify) for browserify ' +
      'or setting mode to production in webpack (https://webpack.js.org/concepts/mode/) ' +
      'to ensure you have the correct code for your production build.'
  )
}

export {
  createStore,
  combineReducers,
  bindActionCreators,
  applyMiddleware,
  compose,
  __DO_NOT_USE__ActionTypes
}

```
`index`入口文件对外释放了我们常用的一些`API`，这个文件本身自带一个`isCrushed`方法，通过注释可以了解到这个被`Redux`开发者称为`dummy function`的方法用于判断`Redux`代码是否被压缩，在开发环境会给出警告，其实是一个压缩检验函数。

#### createStore

通常我们创建一个`store`一般会这么写：

```javascript
let store
store = createStore(rootReducer，initState，compose(applyMiddleware(sagaMiddleware)))
```
第一个参数是返回下一组状态树的函数，通常我们会使用`combineReducers`方法来生成，紧接着我们需要传入初始化的`state`，最后一个参数是`enhancer`，它可以让我们用一些中间件来增强`Store`，比如提供`time travel/persistence`，下面通过整体代码来看一下：

```javascript
import $$observable from 'symbol-observable'

import ActionTypes from './utils/actionTypes'
import isPlainObject from './utils/isPlainObject'

/**
 * Creates a Redux store that holds the state tree.
 * The only way to change the data in the store is to call `dispatch()` on it.
 *
 * There should only be a single store in your app. To specify how different
 * parts of the state tree respond to actions, you may combine several reducers
 * into a single reducer function by using `combineReducers`.
 *
 * @param {Function} reducer A function that returns the next state tree, given
 * the current state tree and the action to handle.
 *
 * @param {any} [preloadedState] The initial state. You may optionally specify it
 * to hydrate the state from the server in universal apps, or to restore a
 * previously serialized user session.
 * If you use `combineReducers` to produce the root reducer function, this must be
 * an object with the same shape as `combineReducers` keys.
 *
 * @param {Function} [enhancer] The store enhancer. You may optionally specify it
 * to enhance the store with third-party capabilities such as middleware,
 * time travel, persistence, etc. The only store enhancer that ships with Redux
 * is `applyMiddleware()`.
 *
 * @returns {Store} A Redux store that lets you read the state, dispatch actions
 * and subscribe to changes.
 */
export default function createStore(reducer, preloadedState, enhancer) {
  // 传参判断
  if (
    (typeof preloadedState === 'function' && typeof enhancer === 'function') ||
    (typeof enhancer === 'function' && typeof arguments[3] === 'function')
  ) {
    throw new Error(
      'It looks like you are passing several store enhancers to ' +
        'createStore(). This is not supported. Instead, compose them ' +
        'together to a single function.'
    )
  }
  // preloadedState默认是undefined
  if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
    enhancer = preloadedState
    preloadedState = undefined
  }

  if (typeof enhancer !== 'undefined') {
    if (typeof enhancer !== 'function') {
      throw new Error('Expected the enhancer to be a function.')
    }
    /// 函数柯里化 传入createStore作为参数
    return enhancer(createStore)(reducer, preloadedState)
  }

  if (typeof reducer !== 'function') {
    throw new Error('Expected the reducer to be a function.')
  }
  // 存储当前的reducer
  let currentReducer = reducer
  // 存储当前的state
  let currentState = preloadedState
  let currentListeners = []
  let nextListeners = currentListeners
  // 判断是否正在分发事件
  let isDispatching = false

  /**
   * This makes a shallow copy of currentListeners so we can use
   * nextListeners as a temporary list while dispatching.
   *
   * This prevents any bugs around consumers calling
   * subscribe/unsubscribe in the middle of a dispatch.
   */
  function ensureCanMutateNextListeners() {
    if (nextListeners === currentListeners) {
      nextListeners = currentListeners.slice()
    }
  }

  /**
   * Reads the state tree managed by the store.
   *
   * @returns {any} The current state tree of your application.
   */
  // 返回当前的state
  function getState() {
    if (isDispatching) {
      throw new Error(
        'You may not call store.getState() while the reducer is executing. ' +
          'The reducer has already received the state as an argument. ' +
          'Pass it down from the top reducer instead of reading it from the store.'
      )
    }

    return currentState
  }

  /**
   * Adds a change listener. It will be called any time an action is dispatched,
   * and some part of the state tree may potentially have changed. You may then
   * call `getState()` to read the current state tree inside the callback.
   *
   * You may call `dispatch()` from a change listener, with the following
   * caveats:
   *
   * 1. The subscriptions are snapshotted just before every `dispatch()` call.
   * If you subscribe or unsubscribe while the listeners are being invoked, this
   * will not have any effect on the `dispatch()` that is currently in progress.
   * However, the next `dispatch()` call, whether nested or not, will use a more
   * recent snapshot of the subscription list.
   *
   * 2. The listener should not expect to see all state changes, as the state
   * might have been updated multiple times during a nested `dispatch()` before
   * the listener is called. It is, however, guaranteed that all subscribers
   * registered before the `dispatch()` started will be called with the latest
   * state by the time it exits.
   *
   * @param {Function} listener A callback to be invoked on every dispatch.
   * @returns {Function} A function to remove this change listener.
   */
  function subscribe(listener) {
    if (typeof listener !== 'function') {
      throw new Error('Expected the listener to be a function.')
    }

    if (isDispatching) {
      throw new Error(
        'You may not call store.subscribe() while the reducer is executing. ' +
          'If you would like to be notified after the store has been updated, subscribe from a ' +
          'component and invoke store.getState() in the callback to access the latest state. ' +
          'See https://redux.js.org/api-reference/store#subscribe(listener) for more details.'
      )
    }

    let isSubscribed = true
    // 浅拷贝一份当前的listener
    ensureCanMutateNextListeners()
    nextListeners.push(listener)

    return function unsubscribe() {
      if (!isSubscribed) {
        return
      }

      if (isDispatching) {
        throw new Error(
          'You may not unsubscribe from a store listener while the reducer is executing. ' +
            'See https://redux.js.org/api-reference/store#subscribe(listener) for more details.'
        )
      }

      isSubscribed = false

      ensureCanMutateNextListeners()
      const index = nextListeners.indexOf(listener)
      nextListeners.splice(index, 1)
    }
  }

  /**
   * Dispatches an action. It is the only way to trigger a state change.
   *
   * The `reducer` function, used to create the store, will be called with the
   * current state tree and the given `action`. Its return value will
   * be considered the **next** state of the tree, and the change listeners
   * will be notified.
   *
   * The base implementation only supports plain object actions. If you want to
   * dispatch a Promise, an Observable, a thunk, or something else, you need to
   * wrap your store creating function into the corresponding middleware. For
   * example, see the documentation for the `redux-thunk` package. Even the
   * middleware will eventually dispatch plain object actions using this method.
   *
   * @param {Object} action A plain object representing “what changed”. It is
   * a good idea to keep actions serializable so you can record and replay user
   * sessions, or use the time travelling `redux-devtools`. An action must have
   * a `type` property which may not be `undefined`. It is a good idea to use
   * string constants for action types.
   *
   * @returns {Object} For convenience, the same action object you dispatched.
   *
   * Note that, if you use a custom middleware, it may wrap `dispatch()` to
   * return something else (for example, a Promise you can await).
   */
  function dispatch(action) {
    if (!isPlainObject(action)) {
      throw new Error(
        'Actions must be plain objects. ' +
          'Use custom middleware for async actions.'
      )
    }

    if (typeof action.type === 'undefined') {
      throw new Error(
        'Actions may not have an undefined "type" property. ' +
          'Have you misspelled a constant?'
      )
    }

    if (isDispatching) {
      throw new Error('Reducers may not dispatch actions.')
    }

    try {
      isDispatching = true
      currentState = currentReducer(currentState, action)
    } finally {
      isDispatching = false
    }

    const listeners = (currentListeners = nextListeners)
    for (let i = 0; i < listeners.length; i++) {
      const listener = listeners[i]
      listener()
    }

    return action
  }

  /**
   * Replaces the reducer currently used by the store to calculate the state.
   *
   * You might need this if your app implements code splitting and you want to
   * load some of the reducers dynamically. You might also need this if you
   * implement a hot reloading mechanism for Redux.
   *
   * @param {Function} nextReducer The reducer for the store to use instead.
   * @returns {void}
   */
  // 用作code splitting 或者 dynamical load reducer
  function replaceReducer(nextReducer) {
    if (typeof nextReducer !== 'function') {
      throw new Error('Expected the nextReducer to be a function.')
    }

    currentReducer = nextReducer

    // This action has a similiar effect to ActionTypes.INIT.
    // Any reducers that existed in both the new and old rootReducer
    // will receive the previous state. This effectively populates
    // the new state tree with any relevant data from the old one.
    dispatch({ type: ActionTypes.REPLACE })
  }

  /**
   * Interoperability point for observable/reactive libraries.
   * @returns {observable} A minimal observable of state changes.
   * For more information, see the observable proposal:
   * https://github.com/tc39/proposal-observable
   */
  function observable() {
    const outerSubscribe = subscribe
    return {
      /**
       * The minimal observable subscription method.
       * @param {Object} observer Any object that can be used as an observer.
       * The observer object should have a `next` method.
       * @returns {subscription} An object with an `unsubscribe` method that can
       * be used to unsubscribe the observable from the store, and prevent further
       * emission of values from the observable.
       */
      subscribe(observer) {
        if (typeof observer !== 'object' || observer === null) {
          throw new TypeError('Expected the observer to be an object.')
        }

        function observeState() {
          if (observer.next) {
            observer.next(getState())
          }
        }

        observeState()
        const unsubscribe = outerSubscribe(observeState)
        return { unsubscribe }
      },

      [$$observable]() {
        return this
      }
    }
  }

  // When a store is created, an "INIT" action is dispatched so that every
  // reducer returns their initial state. This effectively populates
  // the initial state tree.
  dispatch({ type: ActionTypes.INIT })

  return {
    dispatch,
    subscribe,
    getState,
    replaceReducer,
    [$$observable]: observable
  }
}

```

`Store`对外提供了4个`API`：
- getState()
- dispatch(action)
- subscribe(listener)
- replaceReducer(nextReducer)

这些`API`在上面都有一一呈现，从上往下看，首先`createStore.js`会对传参进行处理，根据参数类型来判断是否传入了过多的`enhancer`，同时会给出提示我们可以用`compose`来组合多个`enhancer`，然后判断是否传入了`preloadedState`，没有的话默认赋值为`undefined`，最后判断传入的`enhancer`，当传入的类型不是`function`类型直接抛异常，如果传参类型无误，将`createStore`传入`enhancer`作为参数，这里其实是一个函数柯里化，`enhancer`会返回一个函数，然后将`reducer`和`preloadedState`作为下一级入参。

接下来看一下主要`API`部分的实现，`getState`其实就是返回当前的`state`，如果判断正在调用`dispatch`会报错：

```javascript
  function getState() {
    if (isDispatching) {
      throw new Error(
        'You may not call store.getState() while the reducer is executing. ' +
          'The reducer has already received the state as an argument. ' +
          'Pass it down from the top reducer instead of reading it from the store.'
      )
    }

    return currentState
  }
```
然后是我们常用的`dispatch`方法：

```javascript
  function dispatch(action) {
    if (!isPlainObject(action)) {
      throw new Error(
        'Actions must be plain objects. ' +
          'Use custom middleware for async actions.'
      )
    }

    if (typeof action.type === 'undefined') {
      throw new Error(
        'Actions may not have an undefined "type" property. ' +
          'Have you misspelled a constant?'
      )
    }

    if (isDispatching) {
      throw new Error('Reducers may not dispatch actions.')
    }

    try {
      isDispatching = true
      currentState = currentReducer(currentState, action)
    } finally {
      isDispatching = false
    }

    const listeners = (currentListeners = nextListeners)
    for (let i = 0; i < listeners.length; i++) {
      const listener = listeners[i]
      listener()
    }

    return action
  }
```
调用`dispatch`时候方法内部会先判断传入的`action`是否是`plain objects`，这个方法也很简单：

```javascript
/**
 * @param {any} obj The object to inspect.
 * @returns {boolean} True if the argument appears to be a plain object.
 */
export default function isPlainObject(obj) {
  if (typeof obj !== 'object' || obj === null) return false

  let proto = obj
  while (Object.getPrototypeOf(proto) !== null) {
    proto = Object.getPrototypeOf(proto)
  }

  return Object.getPrototypeOf(obj) === proto
}
```
然后判断`action`是否含有`type`属性，没有的话会报错，当`isDispatching`标志位为`false`的时候，将这个标志位设为`true`，然后将传入的`action`和`currentState`传入`currentReducer`得到当前的状态快照，完成这件事之后又重置`isDispatching`为`false`，以便执行下一轮的`dispatch`，最后依次执行通过`subscribe`方法订阅的回调。

上面提到的`subscribe`方法平常开发比较少用到，你可以在订阅的回调里面通过`getState`方法获取当前的`state`，看一下相关实现：

```javascript

  function subscribe(listener) {
    if (typeof listener !== 'function') {
      throw new Error('Expected the listener to be a function.')
    }

    if (isDispatching) {
      throw new Error(
        'You may not call store.subscribe() while the reducer is executing. ' +
          'If you would like to be notified after the store has been updated, subscribe from a ' +
          'component and invoke store.getState() in the callback to access the latest state. ' +
          'See https://redux.js.org/api-reference/store#subscribe(listener) for more details.'
      )
    }

    let isSubscribed = true
    // 浅拷贝一份当前的listener
    ensureCanMutateNextListeners()
    nextListeners.push(listener)

    return function unsubscribe() {
      if (!isSubscribed) {
        return
      }

      if (isDispatching) {
        throw new Error(
          'You may not unsubscribe from a store listener while the reducer is executing. ' +
            'See https://redux.js.org/api-reference/store#subscribe(listener) for more details.'
        )
      }

      isSubscribed = false

      ensureCanMutateNextListeners()
      const index = nextListeners.indexOf(listener)
      nextListeners.splice(index, 1)
    }
  }
```
就是简单地浅拷贝一份当前的回调`List`，将订阅的回调加入`nextListeners`，返回一个取消订阅的函数，调用这个函数会将该订阅从`nextListeners`里面删除，可以看一个官方的使用示例：

```javascript
function select(state) {
  return state.some.deep.property
}

let currentValue
function handleChange() {
  let previousValue = currentValue
  currentValue = select(store.getState())

  if (previousValue !== currentValue) {
    console.log(
      'Some deep nested property changed from',
      previousValue,
      'to',
      currentValue
    )
  }
}

const unsubscribe = store.subscribe(handleChange)
unsubscribe()
```

最后一个，也是不常用的方法`replaceReducer `：

```javascript
function replaceReducer(nextReducer) {
    if (typeof nextReducer !== 'function') {
      throw new Error('Expected the nextReducer to be a function.')
    }

    currentReducer = nextReducer

    // This action has a similiar effect to ActionTypes.INIT.
    // Any reducers that existed in both the new and old rootReducer
    // will receive the previous state. This effectively populates
    // the new state tree with any relevant data from the old one.
    dispatch({ type: ActionTypes.REPLACE })
  }
```
这个方法一般用作`code splitting`或者 `dynamical load reducer`，成功替换`reducer`之后会发出一个`type`为`ActionTypes.REPLACE`的`action`。

#### combineReducers

基于 Redux 的应用程序中最常见的 state 结构是一个简单的 JavaScript 对象，它最外层的每个 key 中拥有特定域的数据。类似地，给这种 state 结构写 reducer 的方式是分拆成多个 reducer，拆分之后的 reducer 都是相同的结构（state, action），并且每个函数独立负责管理该特定切片 state 的更新。多个拆分之后的 reducer 可以响应一个 action，在需要的情况下独立的更新他们自己的切片 state，最后组合成新的 state。

这个模式是如此的通用，Redux 提供了 combineReducers 去实现这个模式。这是一个高阶 Reducer 的示例，他接收一个拆分后 reducer 函数组成的对象，返回一个新的 Reducer 函数。

```javascript
export default function combineReducers(reducers) {
  const reducerKeys = Object.keys(reducers)
  const finalReducers = {}
  // 过滤一些不存在的reducer
  for (let i = 0; i < reducerKeys.length; i++) {
    const key = reducerKeys[i]

    if (process.env.NODE_ENV !== 'production') {
      if (typeof reducers[key] === 'undefined') {
        warning(`No reducer provided for key "${key}"`)
      }
    }

    if (typeof reducers[key] === 'function') {
      finalReducers[key] = reducers[key]
    }
  }
  const finalReducerKeys = Object.keys(finalReducers)

  // This is used to make sure we don't warn about the same
  // keys multiple times.
  let unexpectedKeyCache
  if (process.env.NODE_ENV !== 'production') {
    unexpectedKeyCache = {}
  }

  let shapeAssertionError
  try {
    assertReducerShape(finalReducers)
  } catch (e) {
    shapeAssertionError = e
  }

  return function combination(state = {}, action) {
    if (shapeAssertionError) {
      throw shapeAssertionError
    }

    if (process.env.NODE_ENV !== 'production') {
      const warningMessage = getUnexpectedStateShapeWarningMessage(
        state,
        finalReducers,
        action,
        unexpectedKeyCache
      )
      if (warningMessage) {
        warning(warningMessage)
      }
    }

    let hasChanged = false
    const nextState = {}
    for (let i = 0; i < finalReducerKeys.length; i++) {
      const key = finalReducerKeys[i]
      const reducer = finalReducers[key]
      const previousStateForKey = state[key]
      const nextStateForKey = reducer(previousStateForKey, action)
      if (typeof nextStateForKey === 'undefined') {
        const errorMessage = getUndefinedStateErrorMessage(key, action)
        throw new Error(errorMessage)
      }
      nextState[key] = nextStateForKey
      hasChanged = hasChanged || nextStateForKey !== previousStateForKey
    }
    return hasChanged ? nextState : state
  }
}
```
`combineReducers`接受一个对象作为参数，这个参数里面每一个`key`都对应着一个值为`reducer function`的函数，在这个方法里面首先会对传入的`reducers`进行过滤，去掉一些不存在的`reducer`，把过滤后的结果存入`finalReducers`，然后通过`assertReducerShape`规范我们写的`reducer`函数，看一眼这个方法：

```javascript
function assertReducerShape(reducers) {
  Object.keys(reducers).forEach(key => {
    const reducer = reducers[key]
    const initialState = reducer(undefined, { type: ActionTypes.INIT })

    if (typeof initialState === 'undefined') {
      throw new Error(
        `Reducer "${key}" returned undefined during initialization. ` +
          `If the state passed to the reducer is undefined, you must ` +
          `explicitly return the initial state. The initial state may ` +
          `not be undefined. If you don't want to set a value for this reducer, ` +
          `you can use null instead of undefined.`
      )
    }

    if (
      typeof reducer(undefined, {
        type: ActionTypes.PROBE_UNKNOWN_ACTION()
      }) === 'undefined'
    ) {
      throw new Error(
        `Reducer "${key}" returned undefined when probed with a random type. ` +
          `Don't try to handle ${
            ActionTypes.INIT
          } or other actions in "redux/*" ` +
          `namespace. They are considered private. Instead, you must return the ` +
          `current state for any unknown actions, unless it is undefined, ` +
          `in which case you must return the initial state, regardless of the ` +
          `action type. The initial state may not be undefined, but can be null.`
      )
    }
  })
}
```
这里给每个`reducer`函数传入了一个`type`为`ActionTypes.INIT`的`action`，按理说正常情况我们的`reducer`在接受到未定义`type`的`action`时候应该返回默认`state`，所以这里判断如果返回了`undefined`那么说明这个`reducer`写的不够规范。

最后返回一个名为`combination`的函数，这个函数其实在`createStore.js`的`dispatch`方法里面有体现：

```javascript
// ... 省略无关代码
  currentState = currentReducer(currentState, action)
// ...省略无关代码
```
这个函数就是把所有的`reducer`循环执行，然后根据传参前后的`state`判断是否改变来决定返回下一组状态还是当前状态。

#### compose

`createStore`的第三个函数就是`store enhancer`用于增强`Store`，当你需要传入多个`store enhancer`的时候就需要通过`compose`方法将其进行组合，看一下组合的逻辑：

```javascript
/**
 * Composes single-argument functions from right to left. The rightmost
 * function can take multiple arguments as it provides the signature for
 * the resulting composite function.
 *
 * @param {...Function} funcs The functions to compose.
 * @returns {Function} A function obtained by composing the argument functions
 * from right to left. For example, compose(f, g, h) is identical to doing
 * (...args) => f(g(h(...args))).
 */

export default function compose(...funcs) {
  if (funcs.length === 0) {
    return arg => arg
  }

  if (funcs.length === 1) {
    return funcs[0]
  }

  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}
```
这个函数会将传入的参数进行组合从右到左进行执行，所有传递给`compose`方法的参数都必须是函数，通过`reduce`方法组合，给用`compose`方法组合后的函数传入的第一个参数会作为`compose`函数最后一个函数参数的参数，联想到我们使用`createStore`方法的时候：

```javascript
enhancer(createStore)(reducer, preloadedState)
```
如果有多个`store enhancer`这里的`enhancer`就是`compose`，`createStore`将作为最后一个`store enhancer`函数的参数，然后按照洋葱圈模型从右向左执行，下面我们可以聊一聊`store enhancer`。

#### store enhancer

> store enhancer，可以翻译成store的增强器，顾名思义，就是增强store的功能。一个store enhancer，实际上就是一个高阶函数，它的参数是创建store的函数（store creator），返回值是一个可以创建功能更加强大的store的函数(enhanced store creator)，这和React中的高阶组件的概念很相似。

一般来说一个`store enhancer`的代码结构如下：
```javascript
function enhancerCreator() {
  return createStore => (...args) => {
    // do something based old store
    // return a new enhanced store
  }
}

```

我们可以找一个`store enhancer`，刚好`redux`内部就有一个用于增强作用于`dispatch`方法的中间件的`store enhancer`生成器，也就是`applyMiddleware`，这里有一点要注意的是，因为`middleware`都是异步的，所以通过这个方法生成的`store enhancer`必须放在第一位，看一下相关实现：

```javascript
import compose from './compose'

/**
 * Creates a store enhancer that applies middleware to the dispatch method
 * of the Redux store. This is handy for a variety of tasks, such as expressing
 * asynchronous actions in a concise manner, or logging every action payload.
 *
 * See `redux-thunk` package as an example of the Redux middleware.
 *
 * Because middleware is potentially asynchronous, this should be the first
 * store enhancer in the composition chain.
 *
 * Note that each middleware will be given the `dispatch` and `getState` functions
 * as named arguments.
 *
 * @param {...Function} middlewares The middleware chain to be applied.
 * @returns {Function} A store enhancer applying the middleware.
 */
export default function applyMiddleware(...middlewares) {
  return createStore => (...args) => {
    // 创建一个新的store
    const store = createStore(...args)
    let dispatch = () => {
      throw new Error(
        'Dispatching while constructing your middleware is not allowed. ' +
          'Other middleware would not be applied to this dispatch.'
      )
    }

    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args)
    }
    // 传入getState和dispatch，返回以next为参数的匿名函数
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}

```
通过传入的`createStore`以及`reducer`和`state`创建一个新的`Store`，然后定义一个临时`dispatch`方法，在通过`applyMiddleWare`处理之前调用`dispatch`会报错，然后会调用`middlewares`数组中每一个中间件函数，我们可以找一个中间件函数看一下内部的处理过程，这里看一下`redux-thunk`的处理方式：

```javascript
function createThunkMiddleware(extraArgument) {
  return ({ dispatch, getState }) => next => action => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }

    return next(action);
  };
}

const thunk = createThunkMiddleware();
thunk.withExtraArgument = createThunkMiddleware;

export default thunk;

```
每一个`middleware`都会接受`Store`的`dispatch`和`getState`方法作为参数，`redux-thunk`的增强型作用是可以让你`dispatch`一个函数，这样就知道了我们的`chain`里面的函数都是类似于这样的：

```javascript
// 省略代码
  return next => action => {
// 省略代码
```
变相的去掉了一层，然后通过`compose`方法组合出来一个增强后的`dispatch`：

```javascript
dispatch = compose(...chain)(store.dispatch)
```
将`store`原生的`dispatch`作为参数，后续的每一个`middleware`都对这个`dispatch`进行增强，当遇到`redux-thunk`的时候，他会检测当前的`action`是否是一个函数，是的话就执行这个函数，同理其他的`middleware`也是依次执行的。

#### 总结

虽然`Redux`是一个体小精悍的库，但它相关的内容和`API`都是精挑细选的，足以衍生出丰富的工具集和可扩展的生态系统。
