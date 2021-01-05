不管是`Vue`框架还是`React`框架，在实际开发使用的过程中我们都会有很多情况下都会有状态共享的需求，这些状态共享会发生在父子组件和兄弟组件之间，我们为了维护这些状态经常会写很多非必要性的代码，这些代码一多起来，维护就会变得很困难，正是由于有这种需求，人们开发了许多相关的库，从`Flux`到`Redux`再到`Vuex`，这些库的大致思路都是：将共享的状态抽离出来，通过定义和隔离状态管理中的各种概念并强制遵守一定的规则，达到代码结构化和易维护的目的。

#### 简单状态管理起步
在说到`Vuex`之前，我们可以先了解下简单的状态管理，也就是**Store模式**

![Store模式](https://user-images.githubusercontent.com/11991572/54919628-407bd480-4f3c-11e9-9b15-af5868589263.png)

当你有一处需要多个实例共享的状态，可以简单地通过维护一份数据来实现共享：

```javascript
const sourceOfTruth = {}

const vmA = new Vue({
  data: sourceOfTruth
})

const vmB = new Vue({
  data: sourceOfTruth
})
```

现在当 `sourceOfTruth` 发生变化，`vmA`和`vmB`都将自动的更新引用它们的视图，但是采用这种方式，在任何时间，我们应用中的任何部分，在任何数据改变后，都不会留下变更过的记录。为了解决这个问题，我们可以采用一个简单的**Store模式**:

```javascript
var store = {
  debug: true,
  state: {
    message: 'Hello!'
  },
  setMessageAction (newValue) {
    if (this.debug) console.log('setMessageAction triggered with', newValue)
    this.state.message = newValue
  },
  clearMessageAction () {
    if (this.debug) console.log('clearMessageAction triggered')
    this.state.message = ''
  }
}
```
所有 store 中 state 的改变，都放置在 store 自身的 action 中去管理。这种集中式状态管理能够被更容易地理解哪种类型的 mutation 将会发生，以及它们是如何被触发。当错误出现时，我们现在也会有一个 log 记录 bug 之前发生了什么，此外，每个实例/组件仍然可以拥有和管理自己的私有状态：

```javascript
var vmA = new Vue({
  data: {
    privateState: {},
    sharedState: store.state
  }
})

var vmB = new Vue({
  data: {
    privateState: {},
    sharedState: store.state
  }
})
```
接着我们继续延伸约定，组件不允许直接修改属于 store 实例的 state，而应执行 action 来分发 (dispatch) 事件通知 store 去改变，这样的话，一个`Flux`架构就实现了。

#### 目录结构

> Vuex 的版本是 3.1.0

```
目录结构

├── helpers.js  提供action、mutations以及getters的查找API
├── index.esm.js
├── index.js  是源码主入口文件，提供store的各module构建安装
├── mixin.js  提供了store在Vue实例上的装载注入
├── module  提供module对象与module对象树的创建功能
│   ├── module-collection.js
│   └── module.js
├── plugins  提供开发辅助插件，如“时光穿梭”功能
│   ├── devtool.js
│   └── logger.js
├── store.js  构建store
└── util.js  提供了工具方法如find、deepCopy、forEachValue以及assert等方法。
```
一般我看源码都是从入口文件开始看，这里是`index.js`：

```javascript
import { Store, install } from './store'
import { mapState, mapMutations, mapGetters, mapActions, createNamespacedHelpers } from './helpers'

export default {
  Store,
  install,
  version: '__VERSION__',
  mapState,
  mapMutations,
  mapGetters,
  mapActions,
  createNamespacedHelpers
}
```
入口文件对外暴露了`Vuex`相关的`API`，至于为什么这里有一个`install`，其实是因为`Vuex`是被当做`Vue`的插件来使用的，开发一个`Vue`插件的话就需要对外暴露一个`install`方法，这个方法第一个参数`Vue`构造器，第二个参数是一个可选的选项对象。

#### Store.js

`Store.js`对外暴露出了`Store`这个类和`install`这个方法，在开始分析`Store.js`之前我们先看一下`install`：

```javascript
export function install (_Vue) {
  if (Vue && _Vue === Vue) {
    // 报错，已经使用了 Vue.use(Vuex)方法注册了
    if (process.env.NODE_ENV !== 'production') {
      console.error(
        '[vuex] already installed. Vue.use(Vuex) should be called only once.'
      )
    }
    return
  }
  Vue = _Vue
  applyMixin(Vue)
}
```

逻辑很简单，只是把传入的`_Vue`赋值给`Vue`，然后调用`applyMixin(Vue)`方法，这个方法定义在`src/mixin.js`中：

```javascript
export default function (Vue) {
  const version = Number(Vue.version.split('.')[0])
  // 在全局beforeCreate钩子里面初始化vuex
  if (version >= 2) {
    Vue.mixin({ beforeCreate: vuexInit })
  } else {
    // override init and inject vuex init procedure
    // for 1.x backwards compatibility.
    const _init = Vue.prototype._init
    Vue.prototype._init = function (options = {}) {
      options.init = options.init
        ? [vuexInit].concat(options.init)
        : vuexInit
      _init.call(this, options)
    }
  }

  /**
   * Vuex init hook, injected into each instances init hooks list.
   */

  function vuexInit () {
    const options = this.$options
    // store injection
    if (options.store) {
      // new Vue({
      //   el: '#app',
      //   router,
      //   store,
      //   render: h => h(App)
      // })
      // 这里获取的store就是上面初始化Vue的时候传入的store，所以后面我们可以通过this.$store获取到store实例
      this.$store = typeof options.store === 'function'
        ? options.store()
        : options.store
    } else if (options.parent && options.parent.$store) {
      // 如果是子组件，则从根组件获取store
      this.$store = options.parent.$store
    }
  }
}

```
`applyMixin`方法主要是为了在`beforeCreate`的全局钩子给所有子组件注入`$store`属性方便后续调用，设置到`this`上所以后面再全局都可以通过`this.$store`来访问`store`对象

#### Store的实例化

当我们引入`Vuex`之后下一步的操作就是实例化一个`Store`对象，会返回一个`store`实例并且传入`Vue`的构造器：

```javascript
const store = new Vuex.Store({
  state: {
    count: 0
  },
  mutations: {
    increment (state) {
      state.count++
    }
  }
})

new Vue({
  el: '#app',
  router,
  store,
  render: h => h(App)
})

```
在`Store`的构造函数当中，首先会对执行环境进行断言(是否调用了Vue.use(Vuex)来初始化/是否支持Promise等)：

```javascript
export class Store {
  constructor (options = {}) {
    // Auto install if it is not done yet and `window` has `Vue`.
    // To allow users to avoid auto-installation in some cases,
    // this code should be placed here. See #731
    // 在浏览器环境下，如果插件还未安装则它会自动安装。
    // 它允许用户在某些情况下避免自动安装。
    if (!Vue && typeof window !== 'undefined' && window.Vue) {
      install(window.Vue)
    }

    if (process.env.NODE_ENV !== 'production') {
      assert(Vue, `must call Vue.use(Vuex) before creating a store instance.`)
      assert(typeof Promise !== 'undefined', `vuex requires a Promise polyfill in this browser.`)
      assert(this instanceof Store, `store must be called with the new operator.`)
    }
    // 省略无关代码
}
```
`assert`这个方法被定义在`utils.js`当中：

```javascript
export function assert (condition, msg) {
  if (!condition) throw new Error(`[vuex] ${msg}`)
}
```
其实这个方法的作用就是抛出一些异常信息，紧接着定义了一些`Store`的内部变量：

 ```javascript
const {
      // 一个数组，包含应用在 store 上的插件方法
      plugins = [],
      // 使 Vuex store 进入严格模式，在严格模式下，任何 mutation 处理函数以外修改 Vuex state 都会抛出错误
      strict = false
    } = options
    // store internal state
    // 判断是否是通过mutation更改的state
    this._committing = false
    // 存放action
    this._actions = Object.create(null)
    this._actionSubscribers = []
    // 存放mutations
    this._mutations = Object.create(null)
    // 存放getters
    this._wrappedGetters = Object.create(null)
    // 传入的options对象，其实就是初始化时候传入的对象 new Vuex.Store({options})
    // {
    //   modules: {
    //     cart,
    //         products
    //   },
    //   strict: debug,
    //       plugins: debug ? [createLogger()] : []
    // })
    // 初始化modules
    this._modules = new ModuleCollection(options)
    // 根据namespace来map对应的module
    this._modulesNamespaceMap = Object.create(null)
    this._subscribers = []
    // 用$watch监测store数据的变化
    this._watcherVM = new Vue()
```
这里有个很有意思的知识点，就是我们发现这里创建空对象的时候用的都是`Object.create(null)`，这是因为如果直接用一个`{}`赋值的话等价于`Object.create(Object.prototype)`，它还会从`Object.prototype`上继承一些方法如`hasOwnProperty`、`isPrototypeOf`等，如果用`Object.create(null)`则说明这个对象的原型是`null`也就是没有继承任何对象。
除此之外，在`Store`的初始化过程中还有几个主要的方法，下面进行逐一的分析：

#### 模块的初始化

由于`Store`使用的是单一的状态树，应用的所有状态会集中到一个比较大的对象。当应用变得非常复杂时，store 对象就有可能变得相当臃肿。为了解决以上问题，Vuex 允许我们将 store 分割成模块（module）。每个模块拥有自己的`state`、`mutation`、`action`、`getter`、甚至是嵌套子模块——从上至下进行同样方式的分割：

```javascript
const moduleA = {
  state: { ... },
  mutations: { ... },
  actions: { ... },
  getters: { ... }
}

const moduleB = {
  state: { ... },
  mutations: { ... },
  actions: { ... },
  getters: { ... },
}

const store = new Vuex.Store({
  modules: {
    a: moduleA,
    b: moduleB
  }
})

store.state.a // -> moduleA 的状态
store.state.b // -> moduleB 的状态
```

完成这种树形结构的构建入口就是：

```javascript
// 传入的options对象，其实就是初始化时候传入的对象 new Vuex.Store({options})
// {
//   modules: {
//     cart,
//         products
//   },
//   strict: debug,
//       plugins: debug ? [createLogger()] : []
// })
this._modules = new ModuleCollection(options)
```
`ModuleCollection`这个类的定义在`src/module/module-collection.js`：

```javascript
import Module from './module'
import { assert, forEachValue } from '../util'

export default class ModuleCollection {
  constructor (rawRootModule) {
    // register root module (Vuex.Store options)
    this.register([], rawRootModule, false)
  }

  // 获取对应于path的Module
  get (path) {
    // this.root根Module
    return path.reduce((module, key) => {
      return module.getChild(key)
    }, this.root)
  }

  getNamespace (path) {
    let module = this.root
    return path.reduce((namespace, key) => {
      module = module.getChild(key)
      return namespace + (module.namespaced ? key + '/' : '')
    }, '')
  }

  update (rawRootModule) {
    update([], this.root, rawRootModule)
  }

  register (path, rawModule, runtime = true) {
    if (process.env.NODE_ENV !== 'production') {
      // 检测module对应的函数的形式是否正确
      assertRawModule(path, rawModule)
    }
    // 构建Module对象
    const newModule = new Module(rawModule, runtime)
    if (path.length === 0) {
      this.root = newModule
    } else {
      // 获取Parent Module
      const parent = this.get(path.slice(0, -1))
      // 添加子Module
      parent.addChild(path[path.length - 1], newModule)
    }

    // register nested modules
    if (rawModule.modules) {
      // 递归注册子Module
      forEachValue(rawModule.modules, (rawChildModule, key) => {
        this.register(path.concat(key), rawChildModule, runtime)
      })
    }
  }

  unregister (path) {
    const parent = this.get(path.slice(0, -1))
    const key = path[path.length - 1]
    if (!parent.getChild(key).runtime) return

    parent.removeChild(key)
  }
}

function update (path, targetModule, newModule) {
  if (process.env.NODE_ENV !== 'production') {
    assertRawModule(path, newModule)
  }

  // update target module
  targetModule.update(newModule)

  // update nested modules
  if (newModule.modules) {
    for (const key in newModule.modules) {
      if (!targetModule.getChild(key)) {
        if (process.env.NODE_ENV !== 'production') {
          console.warn(
            `[vuex] trying to add a new module '${key}' on hot reloading, ` +
            'manual reload is needed'
          )
        }
        return
      }
      update(
        path.concat(key),
        targetModule.getChild(key),
        newModule.modules[key]
      )
    }
  }
}

const functionAssert = {
  assert: value => typeof value === 'function',
  expected: 'function'
}

const objectAssert = {
  assert: value => typeof value === 'function' ||
    (typeof value === 'object' && typeof value.handler === 'function'),
  expected: 'function or object with "handler" function'
}

// actions使用objectAssert是因为在带命名空间的模块注册全局action时action的定义会放在函数handler中
const assertTypes = {
  getters: functionAssert,
  mutations: functionAssert,
  actions: objectAssert
}

function assertRawModule (path, rawModule) {
  Object.keys(assertTypes).forEach(key => {
    if (!rawModule[key]) return
    const assertOptions = assertTypes[key]
    // 循环getters/mutations/actions
    // value和type对应函数体和函数名
    forEachValue(rawModule[key], (value, type) => {
      assert(
        assertOptions.assert(value),
        makeAssertionMessage(path, key, type, value, assertOptions.expected)
      )
    })
  })
}

function makeAssertionMessage (path, key, type, value, expected) {
  let buf = `${key} should be ${expected} but "${key}.${type}"`
  if (path.length > 0) {
    buf += ` in module "${path.join('.')}"`
  }
  buf += ` is ${JSON.stringify(value)}.`
  return buf
}

```

实例化`ModuleCollection`其实就是执行`register`方法，这个方法接受3个参数，其中`path`参数就是`module`的路径，这个值是我们拆分`module`时候`module`的`key`组成的一个数组，以上面为例的话，`moduleA`和`moduleB`的`path`分别为`["a"]`和`["b"]`，如果他们还有子`module`则子`module`的`path`的形式大致如`["a"，"a1"]`/`["b"，"b1"]`，第二个参数其实是定义`module`的配置，像`rawRootModule`就是我们构建一个`Store`的时候传入的那个对象，第三个参数`runtime`表示是否是一个运行时创建的`module`，紧接着在`register`方法内部通过`assertRawModule`方法遍历`module`内部的`getters`、`mutations`、`actions`是否符合要求，紧接着通过`const newModule = new Module(rawModule, runtime)`构建一个`module`对象，看一眼`module`类的实现：

```javascript
import { forEachValue } from '../util'

// Base data struct for store's module, package with some attribute and method
export default class Module {
  constructor (rawModule, runtime) {
    this.runtime = runtime
    // Store some children item
    this._children = Object.create(null)
    // Store the origin module object which passed by programmer
    this._rawModule = rawModule
    const rawState = rawModule.state
    // Store the origin module's state
    //   state() {
    //       return {
    //           // state here instead
    //       }
    //   }
    this.state = (typeof rawState === 'function' ? rawState() : rawState) || {}
  }

  get namespaced () {
    return !!this._rawModule.namespaced
  }

  addChild (key, module) {
    this._children[key] = module
  }

  removeChild (key) {
    delete this._children[key]
  }

  getChild (key) {
    return this._children[key]
  }

  update (rawModule) {
    this._rawModule.namespaced = rawModule.namespaced
    if (rawModule.actions) {
      this._rawModule.actions = rawModule.actions
    }
    if (rawModule.mutations) {
      this._rawModule.mutations = rawModule.mutations
    }
    if (rawModule.getters) {
      this._rawModule.getters = rawModule.getters
    }
  }

  forEachChild (fn) {
    forEachValue(this._children, fn)
  }

  forEachGetter (fn) {
    if (this._rawModule.getters) {
      forEachValue(this._rawModule.getters, fn)
    }
  }

  forEachAction (fn) {
    if (this._rawModule.actions) {
      forEachValue(this._rawModule.actions, fn)
    }
  }

  forEachMutation (fn) {
    if (this._rawModule.mutations) {
      forEachValue(this._rawModule.mutations, fn)
    }
  }
}

```

只是简单地描述了构建出来的每个模块的一些属性和方法，回到上面的`register`函数，构建完`Module`之后，我们先判断`path`的长度，如果长度为0说明是根`module`，将它赋值给`this.root`，否则的话获取到这个`module`的`parent`，然后通过`module`的`addChild`方法建立模块之间的父子关系：

```javascript
// 获取Parent Module
const parent = this.get(path.slice(0, -1))
// 添加子Module
parent.addChild(path[path.length - 1], newModule)
```
这里调用了`get`方法，传入的`path`是`parent`模块的`path`：

```javascript
get (path) {
    // this.root根Module
    return path.reduce((module, key) => {
      return module.getChild(key)
    }, this.root)
  }
```
因为`path`是整个模块树的路径，这里通过`reduce`方法一层层解析去找到对应模块，查找的过程是用的`module.getChild(key)`方法，返回的是`this._children[key]`，这些`_children`就是通过执行`parent.addChild(path[path.length - 1], newModule)`方法添加的，就这样，每一个模块都通过`path`去寻找到`parent``module`，然后通过`addChild`建立父子关系，逐级递进，构建完成整个`module`树。

#### 模块的安装
接下来回到`Store.js`，初始化`modules`之后会执行一些`bind`操作：

```javascript
    const store = this
    const { dispatch, commit } = this
    this.dispatch = function boundDispatch (type, payload) {
      return dispatch.call(store, type, payload)
    }
    this.commit = function boundCommit (type, payload, options) {
      return commit.call(store, type, payload, options)
    }
```
这是为了当我们在组件内部使用`this.$store.commit/this.$store.dispatch`方法时候的`this`指向的是当前的`store`而不是组件本身，这里先略过`commit`和`dispatch`方法的实现，先分析`Store.js`的初始化操作，在初始化`module`之后会进行`module`安装的一些操作：

```javascript
const state = this._modules.root.state
// init root module.
// this also recursively registers all sub-modules
// and collects all module getters inside this._wrappedGetters
installModule(this, state, [], this._modules.root)
```
看一眼`installModule`方法的实现：

```javascript
function installModule (store, rootState, path, module, hot) {
  // 判断是否是根module
  const isRoot = !path.length
  // 获取module的命名空间
  const namespace = store._modules.getNamespace(path)

  // 将module的命名空间和module本身一一对应存储在_modulesNamespaceMap对象里面方便后续查找
  if (module.namespaced) {
    store._modulesNamespaceMap[namespace] = module
  }

  // set state
  if (!isRoot && !hot) {
    const parentState = getNestedState(rootState, path.slice(0, -1))
    const moduleName = path[path.length - 1]
    store._withCommit(() => {
      Vue.set(parentState, moduleName, module.state)
    })
  }

  const local = module.context = makeLocalContext(store, namespace, path)

  module.forEachMutation((mutation, key) => {
    const namespacedType = namespace + key
    registerMutation(store, namespacedType, mutation, local)
  })

  module.forEachAction((action, key) => {
    const type = action.root ? key : namespace + key
    const handler = action.handler || action
    registerAction(store, type, handler, local)
  })

  module.forEachGetter((getter, key) => {
    const namespacedType = namespace + key
    registerGetter(store, namespacedType, getter, local)
  })

  module.forEachChild((child, key) => {
    installModule(store, rootState, path.concat(key), child, hot)
  })
}
```
这个方法接受5个参数：`store`意味着`root store`、`state`就是`root state`、`path`是当前模块路径、`module`是当前模块，`hot`表示是否是热更新。
首先用一个变量`isRoot`判断是否是根`module`，然后通过`ModuleCollection`类的`getNamespace`获取当前路径的命名空间，这里提一下`Vuex`里面命名空间的概念：

> 默认情况下，模块内部的 action、mutation 和 getter 是注册在全局命名空间的——这样使得多个模块能够对同一 mutation 或 action 作出响应。如果希望你的模块具有更高的封装度和复用性，你可以通过添加 namespaced: true 的方式使其成为带命名空间的模块。当模块被注册后，它的所有 getter、action 及 mutation 都会自动根据模块注册的路径调整命名。
例如：

```javascript
const store = new Vuex.Store({
  modules: {
    account: {
      namespaced: true,

      // 模块内容（module assets）
      state: { ... }, // 模块内的状态已经是嵌套的了，使用 `namespaced` 属性不会对其产生影响
      getters: {
        isAdmin () { ... } // -> getters['account/isAdmin']
      },
      actions: {
        login () { ... } // -> dispatch('account/login')
      },
      mutations: {
        login () { ... } // -> commit('account/login')
      },

      // 嵌套模块
      modules: {
        // 继承父模块的命名空间
        myPage: {
          state: { ... },
          getters: {
            profile () { ... } // -> getters['account/profile']
          }
        },

        // 进一步嵌套命名空间
        posts: {
          namespaced: true,

          state: { ... },
          getters: {
            popular () { ... } // -> getters['account/posts/popular']
          }
        }
      }
    }
  }
})
```
回到`installModule`方法，我么可以看一下根据`path`来获取`namespace`的方法实现：

```javascript
 getNamespace (path) {
    let module = this.root
    return path.reduce((namespace, key) => {
      module = module.getChild(key)
      return namespace + (module.namespaced ? key + '/' : '')
    }, '')
  }
```
从根`module`开始，通过`reduce`方法沿着`path`一层层查找子`module`，然后如果发现该`module`配置了`namespaced`，就把该`path`拼接到`namespace`后面，最后返回完整的路径。
接下来如果该`module`配置了`namespaced`，则把该`module`的`namespace`和`module`本身一一对应存储到`_modulesNamespaceMap`对象里面方便后续查找：

```javascript
  // 将module的命名空间和module本身一一对应存储在_modulesNamespaceMap对象里面方便后续查找
  if (module.namespaced) {
    store._modulesNamespaceMap[namespace] = module
  }
```
紧接着是非`root module`下的模块`state`初始化逻辑：

```javascript
  // set state
  if (!isRoot && !hot) {
    const parentState = getNestedState(rootState, path.slice(0, -1))
    const moduleName = path[path.length - 1]
    store._withCommit(() => {
      Vue.set(parentState, moduleName, module.state)
    })
  }
```
先通过`getNestedState`获取父模块的`state`，这个方法的实现大同小异，都是通过`reduce`函数一层层查找到子模块的`state`：

```javascript
function getNestedState (state, path) {
  return path.length
    ? path.reduce((state, key) => state[key], state)
    : state
}
```
随后我们拿到子`module`的名称，调用`store`对象的`_withCommit`方法，这个方法里面的函数执行的操作是给父模块的`state`添加一个名字是`module`的名称的响应式属性，看一下这个方法的作用：

```javascript
_withCommit (fn) {
    const committing = this._committing
    this._committing = true
    fn()
    this._committing = committing
  }
```
`_committing`的初始值为`false`，用来判断是否是通过`mutation`来更改的`state`，因为在严格模式下，无论何时发生了状态变更且不是由`mutation`函数引起的，将会抛出错误，在`Store.js`的源码中有相关代码体现这一点：

```javascript

if (store.strict) {
  enableStrictMode(store)
}

function enableStrictMode (store) {
  store._vm.$watch(function () { return this._data.$$state }, () => {
    if (process.env.NODE_ENV !== 'production') {
      assert(store._committing, `do not mutate vuex store state outside mutation handlers.`)
    }
  }, { deep: true, sync: true })
}
```
`store._vm`是一个在`Store.js`内置的`Vue`对象：

```javascript
  store._vm = new Vue({
    data: {
      $$state: state
    },
    computed
  })
```
然后通过`Vue`的实例方法`$watch`监听每一次`state`的变化，通过断言判断当前`state`的变化是否是通过提交一个`mutation`来引起的，不是的话就报错`do not mutate vuex store state outside mutation handlers`。

初始化非`root module`下的`state`之后下一步操作是构造一个当前`module`的上下文环境：

```javascript
const local = module.context = makeLocalContext(store, namespace, path)
```
`makeLocalContext`支持3个参数，分别是`root store`、当前`module`的`namespace`以及当前`module`的`path`：

```javascript
/**
 * make localized dispatch, commit, getters and state
 * if there is no namespace, just use root ones
 */
function makeLocalContext (store, namespace, path) {
  const noNamespace = namespace === ''

  const local = {
    dispatch: noNamespace ? store.dispatch : (_type, _payload, _options) => {
      const args = unifyObjectStyle(_type, _payload, _options)
      const { payload, options } = args
      let { type } = args

      if (!options || !options.root) {
        type = namespace + type
        if (process.env.NODE_ENV !== 'production' && !store._actions[type]) {
          console.error(`[vuex] unknown local action type: ${args.type}, global type: ${type}`)
          return
        }
      }

      return store.dispatch(type, payload)
    },

    commit: noNamespace ? store.commit : (_type, _payload, _options) => {
      const args = unifyObjectStyle(_type, _payload, _options)
      const { payload, options } = args
      let { type } = args

      if (!options || !options.root) {
        type = namespace + type
        if (process.env.NODE_ENV !== 'production' && !store._mutations[type]) {
          console.error(`[vuex] unknown local mutation type: ${args.type}, global type: ${type}`)
          return
        }
      }

      store.commit(type, payload, options)
    }
  }

  // getters and state object must be gotten lazily
  // because they will be changed by vm update
  Object.defineProperties(local, {
    getters: {
      get: noNamespace
        ? () => store.getters
        : () => makeLocalGetters(store, namespace)
    },
    state: {
      get: () => getNestedState(store.state, path)
    }
  })

  return local
}
```
通过一个变量`noNamespace`判断`module`是否配置了`namespaced`属性，然后构建一个`local`对象，这个对象包含了`commit`和`dispatch`方法，两个方法定义过程差不多，以`commit`为例，如果没有`namespaced`属性，这个`commit`直接指向了`store.commit`，否则构建一个函数，这个函数首先会对传入的参数顺序进行格式化，`unifyObjectStyle`方法兼容了载荷和对象风格的两种提交方式：

```javascript
function unifyObjectStyle (type, payload, options) {
  if (isObject(type) && type.type) {
    options = payload
    payload = type
    type = type.type
  }

  if (process.env.NODE_ENV !== 'production') {
    assert(typeof type === 'string', `expects string as the type, but found ${typeof type}.`)
  }

  return { type, payload, options }
}
```
如果是对象风格的提交方式则对参数位置进行调整，然后返回一个调整位置后的对象，如果没有在`commit`方法里面设置`root：true`参数，则将`type`和`namespace`拼接之后执行`commit`方法，如果设置了`root：true`意味着允许在命名空间模块里提交根的`mutation`。

构建完`local`对象后会在`local`对象上定义两个属性`getters`和`state`：

```javascript
  // getters and state object must be gotten lazily
  // because they will be changed by vm update
  Object.defineProperties(local, {
    getters: {
      get: noNamespace
        ? () => store.getters
        : () => makeLocalGetters(store, namespace)
    },
    state: {
      get: () => getNestedState(store.state, path)
    }
  })
```
`state`的获取比较简单，就是根据`root state`和当前`module`的`path`获取该`module`的`state`，我们看`getters`的实现，如果没有`namespace`，直接返回`root store`的`getters`，否则调用`makeLocalGetters`获取对应`namespace`的`getters`，看一下`makeLocalGetters`方法的实现：

```javascript
function makeLocalGetters (store, namespace) {
  const gettersProxy = {}

  const splitPos = namespace.length
  Object.keys(store.getters).forEach(type => {
    // skip if the target getter is not match this namespace
    if (type.slice(0, splitPos) !== namespace) return

    // extract local getter type
    const localType = type.slice(splitPos)

    // Add a port to the getters proxy.
    // Define as getter property because
    // we do not want to evaluate the getters in this time.
    Object.defineProperty(gettersProxy, localType, {
      get: () => store.getters[type],
      enumerable: true
    })
  })

  return gettersProxy
}
```
这个方法对`this.getters`上所有的可玫举属性进行遍历，然后截取`type`的包含`namespace`的部分和传入的`namespace`进行对比，找到后截取`type`里面的后半部分，也就是不包含`namespace`的部分，然后定义了`gettersProxy`的`get`属性并将其返回。

回到`installModule`方法，在完成构建`local`之后，会循环遍历`module`中定义的`mutation`、`action`、`getters`然后执行注册逻辑，这几个操作都差不多，我们看一个`mutation`的：

```javascript
  module.forEachMutation((mutation, key) => {
    const namespacedType = namespace + key
    registerMutation(store, namespacedType, mutation, local)
  })

 forEachMutation (fn) {
    if (this._rawModule.mutations) {
      forEachValue(this._rawModule.mutations, fn)
    }
  }

  export function forEachValue (obj, fn) {
    // 将对象里面的每一项组合成数组
    Object.keys(obj).forEach(key => fn(obj[key], key))
  }

  function registerMutation (store, type, handler, local) {
    const entry = store._mutations[type] || (store._mutations[type] = [])
    entry.push(function wrappedMutationHandler (payload) {
      handler.call(store, local.state, payload)
    })
  }
```
实际上就是给`root store`的`_mutations`对象的对应`type``push`一个处理函数，这个函数调用时候会将`handler`的`this`指向`root store`。
在`installModule`的最后会循环遍历子`module`然后执行子`module`的`installModule`方法：

```javascript
module.forEachChild((child, key) => {
    installModule(store, rootState, path.concat(key), child, hot)
  })
```
至此，`installModule`方法分析完毕

#### 初始化store vm

这是实例化`Store`的最后一步，通过`resetStoreVM`方法初始化`vm`以及注册`_wrappedGetters`，其中有些代码上面分析过，这里先贴出全部相关代码：

```javascript
function resetStoreVM (store, state, hot) {
  const oldVm = store._vm

  // bind store public getters
  store.getters = {}
  const wrappedGetters = store._wrappedGetters
  const computed = {}
  forEachValue(wrappedGetters, (fn, key) => {
    // use computed to leverage its lazy-caching mechanism
    computed[key] = () => fn(store)
    Object.defineProperty(store.getters, key, {
      get: () => store._vm[key],
      enumerable: true // for local getters
    })
  })

  // use a Vue instance to store the state tree
  // suppress warnings just in case the user has added
  // some funky global mixins
  const silent = Vue.config.silent
  Vue.config.silent = true
  store._vm = new Vue({
    data: {
      $$state: state
    },
    computed
  })
  Vue.config.silent = silent

  // enable strict mode for new vm
  if (store.strict) {
    enableStrictMode(store)
  }

  if (oldVm) {
    if (hot) {
      // dispatch changes in all subscribed watchers
      // to force getter re-evaluation for hot reloading.
      store._withCommit(() => {
        oldVm._data.$$state = null
      })
    }
    Vue.nextTick(() => oldVm.$destroy())
  }
}
```
Vuex 允许我们在 store 中定义“getter”（可以认为是 store 的计算属性）。就像计算属性一样，getter 的返回值会根据它的依赖被缓存起来，且只有当它的依赖值发生了改变才会被重新计算。这里首先遍历`wrappedGetters`得到对应的`fn`组成的数组，然后将其定义为一个个计算属性`computed[key] = () => fn(store)`，`fn(store)`其实就是执行了下面的方法：

```javascript
store._wrappedGetters[type] = function wrappedGetter (store) {
    return rawGetter(
      local.state, // local state
      local.getters, // local getters
      store.state, // root state
      store.getters // root getters
    )
  }
```
_随后给`store.getters`新增属性，访问这些属性也就是使用`this.store.getters`的时候拿到的是`key`对应的计算属性的值:

```javascript
Object.defineProperty(store.getters, key, {
      get: () => store._vm[key],
      enumerable: true // for local getters
    })
```
执行这个`getter`对应的函数等价于执行了`computed[key] = () => fn(store)`这个计算属性对应的函数，由于这个函数依赖了`store`，所以根据计算属性的特性在`store`变化的时候这个`getter`也会得到相应的更新。
方法的最后，处理了`hotUpdate`时候的逻辑，即销毁旧的实例：

```javascript
hotUpdate (newOptions) {
    this._modules.update(newOptions)
    resetStore(this, true)
  }

function resetStoreVM (store, state, hot) {
  const oldVm = store._vm
  // ...
  if (oldVm) {
    if (hot) {
      // dispatch changes in all subscribed watchers
      // to force getter re-evaluation for hot reloading.
      store._withCommit(() => {
        oldVm._data.$$state = null
      })
    }
    Vue.nextTick(() => oldVm.$destroy())
  }
}
```
至此，整个`Store.js`的主题初始化流程已经分析完毕，下面分析一写内置函数以及辅助函数的实现

#### 工具函数
更改`Vuex`的`store`中的状态的唯一方法是提交`mutation`。`Vuex`中的`mutation`非常类似于事件：每个`mutation`都有一个字符串的 事件类型 (type) 和 一个 回调函数 (handler)。这个回调函数就是我们实际进行状态更改的地方，并且它会接受 state 作为第一个参数：

```javascript
const store = new Vuex.Store({
  state: {
    count: 1
  },
  mutations: {
    increment (state) {
      // 变更状态
      state.count++
    }
  }
})

```
我们不能直接调用一个`mutation handler`，需要以相应的`type`调用`store.commit`方法：

```javascript
store.commit('increment')

```
看一下`commit`方法相关的实现：

```javascript
commit (_type, _payload, _options) {
    // 将对象风格的commit格式化
    const {
      type,
      payload,
      options
    } = unifyObjectStyle(_type, _payload, _options)

    const mutation = { type, payload }
    // 通过mutation type找到对应的回调函数handler并执行
    const entry = this._mutations[type]
    if (!entry) {
      if (process.env.NODE_ENV !== 'production') {
        console.error(`[vuex] unknown mutation type: ${type}`)
      }
      return
    }
    this._withCommit(() => {
      entry.forEach(function commitIterator (handler) {
        handler(payload)
      })
    })
    // 通知所有的订阅者
    this._subscribers.forEach(sub => sub(mutation, this.state))

    if (
      process.env.NODE_ENV !== 'production' &&
      options && options.silent
    ) {
      console.warn(
        `[vuex] mutation type: ${type}. Silent option has been removed. ` +
        'Use the filter functionality in the Vue-devtools'
      )
    }
  }
```
首先同样是将载荷方式和对象方式的`commit`格式化，然后找到`type`对应的`mutation`，在确保数据更新方式正确的情况下循环执行`mutation`里面的方法，然后所有`mutation`相关的订阅者。

`dispatch`的逻辑要稍微复杂一些，因为通过`dispatch`分发`action`是可以执行异步操作的，然后在`action`内部执行异步操作后再`commit`一个`mutation`：

```javascript
 dispatch (_type, _payload) {
    // 对象风格的dispatch格式化
    const {
      type,
      payload
    } = unifyObjectStyle(_type, _payload)

    const action = { type, payload }
    const entry = this._actions[type]
    if (!entry) {
      if (process.env.NODE_ENV !== 'production') {
        console.error(`[vuex] unknown action type: ${type}`)
      }
      return
    }
    // 从 3.1.0 起，subscribeAction 也可以指定订阅处理函数的被调用时机应该在一个 action 分发之前还是之后 (默认行为是之前)
    try {
      this._actionSubscribers
        .filter(sub => sub.before)
        .forEach(sub => sub.before(action, this.state))
    } catch (e) {
      if (process.env.NODE_ENV !== 'production') {
        console.warn(`[vuex] error in before action subscribers: `)
        console.error(e)
      }
    }

    const result = entry.length > 1
      ? Promise.all(entry.map(handler => handler(payload)))
      : entry[0](payload)

    return result.then(res => {
      try {
        this._actionSubscribers
          .filter(sub => sub.after)
          .forEach(sub => sub.after(action, this.state))
      } catch (e) {
        if (process.env.NODE_ENV !== 'production') {
          console.warn(`[vuex] error in after action subscribers: `)
          console.error(e)
        }
      }
      return res
    })
  }
```
同样是先格式化参数，然后`type`对应的`action`，先判断该`type`是否存在，然后执行`_actionSubscribers`里面在`action`分发之前的回调，这是3.1.0开始有的新功能，可以指定订阅处理函数的被调用时机应该在一个 action 分发之前还是之后 (默认行为是之前)，这个功能多用于插件使用，然后分发`action`，如果同一个`type`的`action`有多个就用`Promise.all`去分发，否则直接传入载荷，在执行分发`action`完毕之后执行`_actionSubscribers`里面在`action`分发之后的回调，至此完成一次`dispatch`。

#### 辅助函数

为了解决重复代码的冗余性，`Vuex`对外提供了一些工具函数，这些工具函数会自动帮我们生成计算属性减少工作量，这些函数都位于`src/helpers.js`目录下，这里可以分析一下`mapState`的实现：

```javascript
/**
 * Reduce the code which written in Vue.js for getting the state.
 * @param {String} [namespace] - Module's namespace
 * @param {Object|Array} states # Object's item can be a function which accept state and getters for param, you can do something for state and getters in it.
 * @param {Object}
 */
export const mapState = normalizeNamespace((namespace, states) => {
  const res = {}
  normalizeMap(states).forEach(({ key, val }) => {
    res[key] = function mappedState () {
      let state = this.$store.state
      let getters = this.$store.getters
      if (namespace) {
        const module = getModuleByNamespace(this.$store, 'mapState', namespace)
        if (!module) {
          return
        }
        state = module.context.state
        getters = module.context.getters
      }
      return typeof val === 'function'
        ? val.call(this, state, getters)
        : state[val]
    }
    // mark vuex getter for devtools
    res[key].vuex = true
  })
  return res
})
```
`mapState`函数可以传递两个参数，第一个参数是可选参数，代表命名空间字符串，对象形式的第二个参数的成员可以是一个函数：

```javascript
mapState(namespace?: string, map: Array<string> | Object<string | function>): Object
```
执行`mapState`其实就是执行`normalizeNamespace`返回的函数，这个函数的作用也很简单：

```javascript

/**
 * Return a function expect two param contains namespace and map. it will normalize the namespace and then the param's function will handle the new namespace and the map.
 * @param {Function} fn
 * @return {Function}
 */
function normalizeNamespace (fn) {
  return (namespace, map) => {
    if (typeof namespace !== 'string') {
      map = namespace
      namespace = ''
    } else if (namespace.charAt(namespace.length - 1) !== '/') {
      namespace += '/'
    }
    return fn(namespace, map)
  }
}
```
判断是否传入了`namespace`参数，没有的话，将`namespace`赋值给`map`参数，如果传了`namespace`则拼接好路径，最后将处理好的`map`作为`states`传入处理函数。这个函数首先会对`states`执行`normalizeMap`处理，这个方法的作用是把我们传入`mapState`的数组/对象统一转换成一个内部元素都是形如`{ key: 'a', val: 1 }`的数组：

```javascript
function normalizeMap (map) {
  return Array.isArray(map)
    ? map.map(key => ({ key, val: key }))
    : Object.keys(map).map(key => ({ key, val: map[key] }))
}
```
然后循环处理数组，结构每一项的`key`和`val`，用一个空对象存储，`key`作为这个空对象`res`的`key`，`key`对应的值是一个名为`mappedState`的函数，在函数内部获取到了`state`、`getters`，然后再判断数组的`val`是否是一个函数，是的话直接调用，并传入`state`和`getters`，否则直接返回`state[val]`。最后将构建好的`res`对象返回，通过对象展开运算符将这个`res`填充到`computed`中。

#### 总结

在`Vuex`的模式下，我们的组件树构成了一个巨大的视图，无论该组件在树的哪个位置，它都可以获取状态或者触发状态更新的行为，通过定义和隔离状态管理中的各种概念并通过强制规则维持视图和状态间的独立性，我们的代码将会变得更结构化且易维护：

![Vuex](https://user-images.githubusercontent.com/11991572/55219071-e2eacf00-523e-11e9-9769-626db78eed40.png)
