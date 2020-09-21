#### 路由的概念
路由这个概念最开始是在后端出现的，以前使用模板引擎开发页面的时候经常会看到这样的路径：
```
http://hometown.xxx.edu.cn/bbs/forum.php
```
有时还会有带.asp或.html的路径，这就是所谓的SSR(Server Side Render)，通过服务端渲染，直接返回页面。

其响应过程是这样的

1.浏览器发出请求

2.服务器监听到80端口（或443）有请求过来，并解析url路径

3.根据服务器的路由配置，返回相应信息（可以是 html 字串，也可以是 json 数据，图片等）

4.浏览器根据数据包的Content-Type来决定如何解析数据

简单来说路由就是用来跟后端服务器进行交互的一种方式，通过不同的路径，来请求不同的资源，请求不同的页面是路由的其中一种功能。就像路由器在网络层中扮演的角色一样，肩负着将数据包正确导向目的地址的重任，只不过在这里变成了客户端浏览器的指路人，所谓的前端路由，指的是一种能力，即：
> 不依赖于服务器，根据不同的URL渲染不同的页面

#### 前端路由与后端路由
在`Ajax`还没有诞生的时候，路由的工作是交给后端来完成的，当进行页面切换的时候，浏览器会发送不同的`URL`请求，服务器接收到浏览器的请求时，通过解析不同的`URL`去拼接需要的`Html`或模板，然后将结果返回到浏览器端进行渲染。

服务器端路由同样是有利亦有弊。它的好处是安全性更高，更严格得控制页面的展现。这在某些场景中是很有用的，譬如下单支付流程，每一步只有在上一步成功执行之后才能抵达。这在服务器端可以为每一步流程添加验证机制，只有验证通过才返回正确的页面。那么前端路由不能实现每一步的验证？自然不是，姑且相信你的代码可以写的很严谨，保证正常情况下流程不会错，但是另一个不得不面对的事实是：前端是毫无安全性可言的。用户可以肆意修改代码来进入不同的流程，你可能会为此添加不少的处理逻辑。相较之下，当然是后端控制页面的进入权限更为安全和简便。

另一方面，后端路由无疑增加了服务器端的负荷，并且需要reload页面，用户体验其实不佳。

#### 前端路由的出现
在 90s 年代初，大多数的网页都是通过直接返回`HTML`的，用户的每次更新操作都需要重新刷新页面。及其影响交互体验，随着网络的发展，迫切需要一种方案来改善这种情况。

1996，微软首先提出 iframe 标签，`iframe`带来了异步加载和请求元素的概念，随后在 1998 年，微软的 Outloook Web App 团队提出`Ajax`的基本概念（XMLHttpRequest的前身），并在`IE5`通过`ActiveX`来实现了这项技术。在微软实现这个概念后，其他浏览器比如`Mozilia`，`Safari`，`Opera`相继以 `XMLHttpRequest`来实现`Ajax`。（😭 兼容问题从此出现，话说微软命名真喜欢用X，MFC源码一大堆。。）不过在 IE7 发布时，微软选择了妥协，兼容了`XMLHttpRequest`的实现。

有了`Ajax`后，用户交互就不用每次都刷新页面，体验带来了极大的提升。

但真正让这项技术发扬光大的，(｡･∀･)ﾉﾞ还是后来的 Google Map，它的出现向人们展现了`Ajax`的真正魅力，释放了众多开发人员的想象力，让其不仅仅局限于简单的数据和页面交互，为后来异步交互体验方式的繁荣发展带来了根基。

而异步交互体验的更高级版本就是我们熟知的`SPA`，`SPA`不单单在页面交互上做到了不刷新，而且在页面之间跳转也做到了不刷新，为了做到这一点，就促使了前端路由的诞生。

#### 前端路由的实现方式
前端路由其实只要解决两个问题：
- 在页面不刷新的前提下实现url变化
- 捕捉到url的变化，以便执行页面替换逻辑
在 2014 年之前，大家是通过 hash 来实现路由，url hash 就是类似于：
```
http://www.xxx.com/#/login

```
这种 #。后面`hash`值的变化，并不会导致浏览器向服务器发出请求，浏览器不发出请求，也就不会刷新页面。另外每次`hash`值的变化，还会触发`hashchange`这个事件，通过这个事件我们就可以知道`hash`值发生了哪些变化。然后我们便可以监听`hashchange`来实现更新页面部分内容的操作：

```javascript
function matchAndUpdate () {
   // todo 匹配 hash 做 dom 更新操作
}
window.addEventListener('hashchange', matchAndUpdate)

```
后来，因为`HTML5`标准发布。多了两个 API，`pushState`和`replaceState`，通过这两个`API`可以改变 `url`地址且不会发送请求。同时还有`popstate`事件。通过这些就能用另一种方式来实现前端路由了，但原理都是跟`hash`实现相同的。用了`HTML5`的实现，单页路由的`url`就不会多出一个#，变得更加美观。但因为没有 # 号，所以当用户刷新页面之类的操作时，浏览器还是会给服务器发送请求。为了避免出现这种情况，所以这个实现需要服务器的支持，需要把所有路由都重定向到根页面：

```javascript
function matchAndUpdate () {
   // todo 匹配路径 做 dom 更新操作
}
window.addEventListener('popstate', matchAndUpdate)

```
#### Vue-Router的实现方式
`Vue-Router`跟`Vuex`一样都是通过`Vue.use`这个全局`API`来注册的，这个方法定义在`vue/src/core/global-api/use.js`：

```javascript
/* @flow */

import { toArray } from '../util/index'

export function initUse (Vue: GlobalAPI) {
  Vue.use = function (plugin: Function | Object) {
    const installedPlugins = (this._installedPlugins || (this._installedPlugins = []))
    if (installedPlugins.indexOf(plugin) > -1) {
      return this
    }

    // additional parameters
    const args = toArray(arguments, 1)
    args.unshift(this)
    if (typeof plugin.install === 'function') {
      plugin.install.apply(plugin, args)
    } else if (typeof plugin === 'function') {
      plugin.apply(null, args)
    }
    installedPlugins.push(plugin)
    return this
  }
}
```
`Vue.use`接受一个`plugin`参数，并且维护了一个`_installedPlugins`数组，它存储所有注册过的`plugin`；如果`plugin`是一个对象，会判断`plugin`有没有定义`install`方法，如果有的话则调用该方法，并且该方法执行的第一个参数是`Vue`；如果`plugin`是一个函数，它会被作为`install`方法，最后把`plugin`存储到`installedPlugins`数组里面，`Vue`的这种插件注册的机制有一个好处就是我们不需要额外的去`import Vue`了。

##### 路由的注册
`Vue-Router`的入口在`src/index.js`，其中`install`方法定义在`src/install.js`，可以看一下`src`下面的目录结构：

```
├── components
│   ├── link.js
│   └── view.js
├── create-matcher.js
├── create-route-map.js
├── history
│   ├── abstract.js
│   ├── base.js
│   ├── hash.js
│   └── html5.js
├── index.js
├── install.js
└── util
    ├── async.js
    ├── dom.js
    ├── location.js
    ├── misc.js
    ├── params.js
    ├── path.js
    ├── push-state.js
    ├── query.js
    ├── resolve-components.js
    ├── route.js
    ├── scroll.js
    └── warn.js

```
简单看下`install`的流程：

```javascript
import View from './components/view'
import Link from './components/link'

export let _Vue

export function install (Vue) {
  // 确保Vue-Router只被install一次
  if (install.installed && _Vue === Vue) return
  install.installed = true

  _Vue = Vue

  const isDef = v => v !== undefined

  const registerInstance = (vm, callVal) => {
    let i = vm.$options._parentVnode
    if (isDef(i) && isDef(i = i.data) && isDef(i = i.registerRouteInstance)) {
      i(vm, callVal)
    }
  }

  Vue.mixin({
    // 在beforeCreate钩子里面初始化路由
    beforeCreate () {
      // 根组件的$options上才有router对象
      if (isDef(this.$options.router)) {
        // 设置根路由
        this._routerRoot = this
        // 获取到根组件上的router实例
        this._router = this.$options.router
        // 路由初始化
        this._router.init(this)
        // 为_route属性实现双向绑定
        Vue.util.defineReactive(this, '_route', this._router.history.current)
      } else {
        // 获取父组件的_routerRoot
        this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
      }
      // 注册<router-view></router-view>实例的钩子
      registerInstance(this, this)
    },
    destroyed () {
      registerInstance(this)
    }
  })
  // 方便全局通过this.$router获取路由实例
  Object.defineProperty(Vue.prototype, '$router', {
    get () { return this._routerRoot._router }
  })
  // 方便全局通过this.$route获取路由对象
  Object.defineProperty(Vue.prototype, '$route', {
    get () { return this._routerRoot._route }
  })
  // 注册全局组件<router-view/>和<router-link/>
  Vue.component('RouterView', View)
  Vue.component('RouterLink', Link)
  // 使用和created相同的合并策略
  const strats = Vue.config.optionMergeStrategies
  // use the same hook merging strategy for route hooks
  strats.beforeRouteEnter = strats.beforeRouteLeave = strats.beforeRouteUpdate = strats.created
}

```
首先通过设立一个`installed`标志位来确保`Vue-Router`只被安装一次，然后通过变量`_Vue`承载传入的`Vue`实例，然后利用`Vue.mixin`向每一个`Vue`实例注册`beforeCreate`和`destroyed`钩子函数。

在`beforeCreate`函数里面，如果是根组件，将根组件赋值给`this._routerRoot`，获取根组件的路由实例之后执行`init`初始化函数，然后调用`Vue`的`defineReactive`将`_route`变为响应式对象，如果不是根组件则获取父组件的`_routerRoot`属性。

在`beforeCreate`函数的最后部分和`destroyed`函数里面都执行了`registerInstance`函数，这个函数是注册`<router-view>`实例的钩子函数，根据传入参数的个数来决定是注册还是取消注册，函数的定义在`src/components/view.js`里面:

```javascript
   // attach instance registration hook
    // this will be called in the instance's injected lifecycle hooks
    data.registerRouteInstance = (vm, val) => {
      // val could be undefined for unregistration
      const current = matched.instances[name]
      if (
        (val && current !== vm) ||
        (!val && current === vm)
      ) {
        matched.instances[name] = val
      }
    }
```
回到`install.js`，紧接着，为了让我们能够全局的使用`this.$router`和`this.$route`在`Vue`原型上定义了对应的`get`方法，然后通过`Vue.component`注册了全局组件`注册全局组件<router-view/>`和`和<router-link/>`，最后定义了一些钩子函数的使用策略，这就是整个`Vue-Router`的安装过程。

##### 路由的实例化
先看一下`Vue-Router`的构造函数，当我们`new`一个`Vue-Router`的时候都干了些什么：

```javascript
/* @flow */

import { install } from './install'
import { START } from './util/route'
import { assert } from './util/warn'
import { inBrowser } from './util/dom'
import { cleanPath } from './util/path'
import { createMatcher } from './create-matcher'
import { normalizeLocation } from './util/location'
import { supportsPushState } from './util/push-state'

import { HashHistory } from './history/hash'
import { HTML5History } from './history/html5'
import { AbstractHistory } from './history/abstract'

import type { Matcher } from './create-matcher'

export default class VueRouter {
  static install: () => void;
  static version: string;
  app: any;
  apps: Array<any>;
  ready: boolean;
  readyCbs: Array<Function>;
  options: RouterOptions;
  mode: string;
  history: HashHistory | HTML5History | AbstractHistory;
  matcher: Matcher;
  fallback: boolean;
  beforeHooks: Array<?NavigationGuard>;
  resolveHooks: Array<?NavigationGuard>;
  afterHooks: Array<?AfterNavigationHook>;

  constructor (options: RouterOptions = {}) {
    // 根Vue实例
    this.app = null
    // 存储含有this.$options.router属性的Vue实例
    this.apps = []
    // 传入路由的配置
    this.options = options
    this.beforeHooks = []
    this.resolveHooks = []
    this.afterHooks = []
    // 创建路由匹配对象
    this.matcher = createMatcher(options.routes || [], this)
    // 默认为hash模式
    let mode = options.mode || 'hash'
    // 当浏览器不支持 history.pushState 控制路由是否应该回退到 hash 模式。默认值为 true
    this.fallback = mode === 'history' && !supportsPushState && options.fallback !== false
    if (this.fallback) {
      mode = 'hash'
    }
    // 支持所有 JavaScript 运行环境，如 Node.js 服务器端。如果发现没有浏览器的 API，路由会自动强制进入这个模式
    if (!inBrowser) {
      mode = 'abstract'
    }
    this.mode = mode
    // 根据mode采用不同的路由方式
    switch (mode) {
      case 'history':
        this.history = new HTML5History(this, options.base)
        break
      case 'hash':
        this.history = new HashHistory(this, options.base, this.fallback)
        break
      case 'abstract':
        this.history = new AbstractHistory(this, options.base)
        break
      default:
        if (process.env.NODE_ENV !== 'production') {
          assert(false, `invalid mode: ${mode}`)
        }
    }
  }

  match (
    raw: RawLocation,
    current?: Route,
    redirectedFrom?: Location
  ): Route {
    return this.matcher.match(raw, current, redirectedFrom)
  }

  get currentRoute (): ?Route {
    return this.history && this.history.current
  }

  init (app: any /* Vue component instance */) {
    // 在初始化Vue-Router之前必须先通过Vue.use(VueRouter)注册
    process.env.NODE_ENV !== 'production' && assert(
      install.installed,
      `not installed. Make sure to call \`Vue.use(VueRouter)\` ` +
      `before creating root instance.`
    )

    this.apps.push(app)

    // set up app destroyed handler
    // https://github.com/vuejs/vue-router/issues/2639
    app.$once('hook:destroyed', () => {
      // clean out app from this.apps array once destroyed
      const index = this.apps.indexOf(app)
      if (index > -1) this.apps.splice(index, 1)
      // ensure we still have a main app or null if no apps
      // we do not release the router so it can be reused
      if (this.app === app) this.app = this.apps[0] || null
    })

    // main app previously initialized
    // return as we don't need to set up new history listener
    if (this.app) {
      return
    }

    this.app = app

    const history = this.history

    if (history instanceof HTML5History) {
      history.transitionTo(history.getCurrentLocation())
    } else if (history instanceof HashHistory) {
      const setupHashListener = () => {
        history.setupListeners()
      }
      history.transitionTo(
        history.getCurrentLocation(),
        setupHashListener,
        setupHashListener
      )
    }

    history.listen(route => {
      this.apps.forEach((app) => {
        app._route = route
      })
    })
  }

  beforeEach (fn: Function): Function {
    return registerHook(this.beforeHooks, fn)
  }

  beforeResolve (fn: Function): Function {
    return registerHook(this.resolveHooks, fn)
  }

  afterEach (fn: Function): Function {
    return registerHook(this.afterHooks, fn)
  }

  onReady (cb: Function, errorCb?: Function) {
    this.history.onReady(cb, errorCb)
  }

  onError (errorCb: Function) {
    this.history.onError(errorCb)
  }

  push (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    this.history.push(location, onComplete, onAbort)
  }

  replace (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    this.history.replace(location, onComplete, onAbort)
  }

  go (n: number) {
    this.history.go(n)
  }

  back () {
    this.go(-1)
  }

  forward () {
    this.go(1)
  }

  getMatchedComponents (to?: RawLocation | Route): Array<any> {
    const route: any = to
      ? to.matched
        ? to
        : this.resolve(to).route
      : this.currentRoute
    if (!route) {
      return []
    }
    return [].concat.apply([], route.matched.map(m => {
      return Object.keys(m.components).map(key => {
        return m.components[key]
      })
    }))
  }

  resolve (
    to: RawLocation,
    current?: Route,
    append?: boolean
  ): {
    location: Location,
    route: Route,
    href: string,
    // for backwards compat
    normalizedTo: Location,
    resolved: Route
  } {
    current = current || this.history.current
    const location = normalizeLocation(
      to,
      current,
      append,
      this
    )
    const route = this.match(location, current)
    const fullPath = route.redirectedFrom || route.fullPath
    const base = this.history.base
    const href = createHref(base, fullPath, this.mode)
    return {
      location,
      route,
      href,
      // for backwards compat
      normalizedTo: location,
      resolved: route
    }
  }

  addRoutes (routes: Array<RouteConfig>) {
    this.matcher.addRoutes(routes)
    if (this.history.current !== START) {
      this.history.transitionTo(this.history.getCurrentLocation())
    }
  }
}

function registerHook (list: Array<any>, fn: Function): Function {
  list.push(fn)
  return () => {
    const i = list.indexOf(fn)
    if (i > -1) list.splice(i, 1)
  }
}

function createHref (base: string, fullPath: string, mode) {
  var path = mode === 'hash' ? '#' + fullPath : fullPath
  return base ? cleanPath(base + '/' + path) : path
}

VueRouter.install = install
VueRouter.version = '__VERSION__'
// 通过link标签引用js的实行自动注册
if (inBrowser && window.Vue) {
  window.Vue.use(VueRouter)
}

```
构造函数里面定义了一些属性，其中`this.app`表示根`Vue`的实例，`this.apps`存储含有`this.$options.router`属性的Vue实例，初始化`Vue-Router`后传入的配置都会存储在`this.options`，` this.beforeHooks`、`this.resolveHooks`、`this.afterHooks`用来存储钩子函数，`this.matcher`是路由匹配后返回的对象，`this.fallback`会根据配置的`mode`参数以及浏览器支持度来决定给是否回退到`hash`模式，`this.mode`就是路由创建的模式，这里提供`hash`、`history`、`abstract`三种模式，`this.history`表示根据不同的路由模式来创建的路由`history`的具体实现方式。

实例化`Vue-Router`之后会返回它的实例`router`，我们在使用`Vue-Router`的时候需要在初始化`Vue`的时候传入这个`router`属性：

```javascript
new Vue({
  el: '#app',
  router,
  render: h => h(App)
})
```
这个时候会把`router`属性配置到`this.$options`，回想到`install.js`里面在`beforeCreate`钩子函数里面执行的方法：

```javascript
    beforeCreate () {
      // 根组件的$options上才有router对象
      if (isDef(this.$options.router)) {
        // 设置根路由
        this._routerRoot = this
        // 获取到根组件上的router实例
        this._router = this.$options.router
        // 路由初始化
        this._router.init(this)
        // ....
      }
```
所以这个时候会执行`init`方法：

```javascript

  init (app: any /* Vue component instance */) {
    // 在初始化Vue-Router之前必须先通过Vue.use(VueRouter)注册
    process.env.NODE_ENV !== 'production' && assert(
      install.installed,
      `not installed. Make sure to call \`Vue.use(VueRouter)\` ` +
      `before creating root instance.`
    )
    // 存储app实例
    this.apps.push(app)

    // set up app destroyed handler
    // https://github.com/vuejs/vue-router/issues/2639
    app.$once('hook:destroyed', () => {
      // clean out app from this.apps array once destroyed
      const index = this.apps.indexOf(app)
      if (index > -1) this.apps.splice(index, 1)
      // ensure we still have a main app or null if no apps
      // we do not release the router so it can be reused
      if (this.app === app) this.app = this.apps[0] || null
    })

    // main app previously initialized
    // return as we don't need to set up new history listener
    if (this.app) {
      return
    }

    this.app = app

    const history = this.history
    // 根据history实现的方式不同执行不同的逻辑
    if (history instanceof HTML5History) {
      history.transitionTo(history.getCurrentLocation())
    } else if (history instanceof HashHistory) {
      const setupHashListener = () => {
        history.setupListeners()
      }
      history.transitionTo(
        history.getCurrentLocation(),
        setupHashListener,
        setupHashListener
      )
    }
    // 更新根组件的路由对象
    history.listen(route => {
      this.apps.forEach((app) => {
        app._route = route
      })
    })
  }
```
`init`其实没干很多事情，首先把传入的`Vue`实例存储到`apps`数组中，然后把`this.history`赋值给一个本地变量，根据`this.history`实现方式的不同执行不同的逻辑，最后通过`history`的回调更新路由对象也就是`this.$route`。

无论`this.history`是基于`history`还是`hash`实现的，最后都会调用`transitionTo`方法，这个方法定义在`src/history/base.js`：

```javascript
transitionTo (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    const route = this.router.match(location, this.current)
    this.confirmTransition(route, () => {
    // ...
  }
```
实际上就是调用`match`方法：

```javascript

  match (
    raw: RawLocation,
    current?: Route,
    redirectedFrom?: Location
  ): Route {
    return this.matcher.match(raw, current, redirectedFrom)
  }
```
那我们可以先把上面的逻辑放一边，先了解一下`matchers`的构建，相关的源码在`src/create-matcher.js`:

##### match的实现
```javascript
/* @flow */

import type VueRouter from './index'
import { resolvePath } from './util/path'
import { assert, warn } from './util/warn'
import { createRoute } from './util/route'
import { fillParams } from './util/params'
import { createRouteMap } from './create-route-map'
import { normalizeLocation } from './util/location'

export type Matcher = {
  match: (raw: RawLocation, current?: Route, redirectedFrom?: Location) => Route;
  addRoutes: (routes: Array<RouteConfig>) => void;
};

export function createMatcher (
  routes: Array<RouteConfig>,
  router: VueRouter
): Matcher {
  const { pathList, pathMap, nameMap } = createRouteMap(routes)
  // 添加路由路径关系映射
  function addRoutes (routes) {
    createRouteMap(routes, pathList, pathMap, nameMap)
  }

  function match (
    raw: RawLocation,
    currentRoute?: Route,
    redirectedFrom?: Location
  ): Route {
    // 根据raw和currentRoute计算出新的location
    const location = normalizeLocation(raw, currentRoute, false, router)
    const { name } = location

    if (name) {
      // 如果是命名路由，取出对应的路由record
      const record = nameMap[name]
      if (process.env.NODE_ENV !== 'production') {
        warn(record, `Route with name '${name}' does not exist`)
      }
      // 生成一条新记录
      if (!record) return _createRoute(null, location)
      const paramNames = record.regex.keys
        .filter(key => !key.optional)
        .map(key => key.name)

      if (typeof location.params !== 'object') {
        location.params = {}
      }
      // 赋值params
      if (currentRoute && typeof currentRoute.params === 'object') {
        for (const key in currentRoute.params) {
          if (!(key in location.params) && paramNames.indexOf(key) > -1) {
            location.params[key] = currentRoute.params[key]
          }
        }
      }

      if (record) {
        location.path = fillParams(record.path, location.params, `named route "${name}"`)
        return _createRoute(record, location, redirectedFrom)
      }
    } else if (location.path) {
      location.params = {}
      for (let i = 0; i < pathList.length; i++) {
        const path = pathList[i]
        const record = pathMap[path]
        if (matchRoute(record.regex, location.path, location.params)) {
          return _createRoute(record, location, redirectedFrom)
        }
      }
    }
    // no match
    return _createRoute(null, location)
  }

  function redirect (
    record: RouteRecord,
    location: Location
  ): Route {
    const originalRedirect = record.redirect
    let redirect = typeof originalRedirect === 'function'
      ? originalRedirect(createRoute(record, location, null, router))
      : originalRedirect

    if (typeof redirect === 'string') {
      redirect = { path: redirect }
    }

    if (!redirect || typeof redirect !== 'object') {
      if (process.env.NODE_ENV !== 'production') {
        warn(
          false, `invalid redirect option: ${JSON.stringify(redirect)}`
        )
      }
      return _createRoute(null, location)
    }

    const re: Object = redirect
    const { name, path } = re
    let { query, hash, params } = location
    query = re.hasOwnProperty('query') ? re.query : query
    hash = re.hasOwnProperty('hash') ? re.hash : hash
    params = re.hasOwnProperty('params') ? re.params : params

    if (name) {
      // resolved named direct
      const targetRecord = nameMap[name]
      if (process.env.NODE_ENV !== 'production') {
        assert(targetRecord, `redirect failed: named route "${name}" not found.`)
      }
      return match({
        _normalized: true,
        name,
        query,
        hash,
        params
      }, undefined, location)
    } else if (path) {
      // 1. resolve relative redirect
      const rawPath = resolveRecordPath(path, record)
      // 2. resolve params
      const resolvedPath = fillParams(rawPath, params, `redirect route with path "${rawPath}"`)
      // 3. rematch with existing query and hash
      return match({
        _normalized: true,
        path: resolvedPath,
        query,
        hash
      }, undefined, location)
    } else {
      if (process.env.NODE_ENV !== 'production') {
        warn(false, `invalid redirect option: ${JSON.stringify(redirect)}`)
      }
      return _createRoute(null, location)
    }
  }

  function alias (
    record: RouteRecord,
    location: Location,
    matchAs: string
  ): Route {
    const aliasedPath = fillParams(matchAs, location.params, `aliased route with path "${matchAs}"`)
    const aliasedMatch = match({
      _normalized: true,
      path: aliasedPath
    })
    if (aliasedMatch) {
      const matched = aliasedMatch.matched
      const aliasedRecord = matched[matched.length - 1]
      location.params = aliasedMatch.params
      return _createRoute(aliasedRecord, location)
    }
    return _createRoute(null, location)
  }

  function _createRoute (
    record: ?RouteRecord,
    location: Location,
    redirectedFrom?: Location
  ): Route {
    if (record && record.redirect) {
      return redirect(record, redirectedFrom || location)
    }
    if (record && record.matchAs) {
      return alias(record, location, record.matchAs)
    }
    return createRoute(record, location, redirectedFrom, router)
  }

  return {
    match,
    addRoutes
  }
}

function matchRoute (
  regex: RouteRegExp,
  path: string,
  params: Object
): boolean {
  const m = path.match(regex)

  if (!m) {
    return false
  } else if (!params) {
    return true
  }

  for (let i = 1, len = m.length; i < len; ++i) {
    const key = regex.keys[i - 1]
    const val = typeof m[i] === 'string' ? decodeURIComponent(m[i]) : m[i]
    if (key) {
      // Fix #1994: using * with props: true generates a param named 0
      params[key.name || 'pathMatch'] = val
    }
  }

  return true
}

function resolveRecordPath (path: string, record: RouteRecord): string {
  return resolvePath(path, record.parent ? record.parent.path : '/', true)
}

```
`createMatcher`接受两个参数，第一个是初始化路由的配置对象`routes`，第二个是我们的路由实例`router`，首先会跑一个`createRouteMap`的逻辑，这个方法的作用是创建一个路由映射，这个方法定义在`src/create-route-map.js`：

```javascript
/* @flow */

import Regexp from 'path-to-regexp'
import { cleanPath } from './util/path'
import { assert, warn } from './util/warn'

export function createRouteMap (
  routes: Array<RouteConfig>,
  oldPathList?: Array<string>,
  oldPathMap?: Dictionary<RouteRecord>,
  oldNameMap?: Dictionary<RouteRecord>
): {
  pathList: Array<string>;
  pathMap: Dictionary<RouteRecord>;
  nameMap: Dictionary<RouteRecord>;
} {
  // the path list is used to control path matching priority
  const pathList: Array<string> = oldPathList || []
  // $flow-disable-line
  const pathMap: Dictionary<RouteRecord> = oldPathMap || Object.create(null)
  // $flow-disable-line
  const nameMap: Dictionary<RouteRecord> = oldNameMap || Object.create(null)

  routes.forEach(route => {
    addRouteRecord(pathList, pathMap, nameMap, route)
  })
  // 确保通配符的路径在最后才被匹配
  // ensure wildcard routes are always at the end
  for (let i = 0, l = pathList.length; i < l; i++) {
    if (pathList[i] === '*') {
      pathList.push(pathList.splice(i, 1)[0])
      l--
      i--
    }
  }

  return {
    pathList,
    pathMap,
    nameMap
  }
}

function addRouteRecord (
  pathList: Array<string>,
  pathMap: Dictionary<RouteRecord>,
  nameMap: Dictionary<RouteRecord>,
  route: RouteConfig,
  parent?: RouteRecord,
  matchAs?: string
) {
  const { path, name } = route
  if (process.env.NODE_ENV !== 'production') {
    // path不能为空并且component的值必须用一个组件名而不是一个string字符串
    assert(path != null, `"path" is required in a route configuration.`)
    assert(
      typeof route.component !== 'string',
      `route config "component" for path: ${String(path || name)} cannot be a ` +
      `string id. Use an actual component instead.`
    )
  }
  // pathToRegexpOptions是编译正则的选项
  const pathToRegexpOptions: PathToRegexpOptions = route.pathToRegexpOptions || {}
  // normalize path
  const normalizedPath = normalizePath(
    path,
    parent,
    pathToRegexpOptions.strict
  )
  // caseSensitive匹配规则是否大小写敏感？(默认值：false)
  if (typeof route.caseSensitive === 'boolean') {
    pathToRegexpOptions.sensitive = route.caseSensitive
  }

  const record: RouteRecord = {
    path: normalizedPath,
    regex: compileRouteRegex(normalizedPath, pathToRegexpOptions),
    components: route.components || { default: route.component },
    instances: {},
    name,
    parent,
    matchAs,
    redirect: route.redirect,
    beforeEnter: route.beforeEnter,
    meta: route.meta || {},
    props: route.props == null
      ? {}
      : route.components
        ? route.props
        : { default: route.props }
  }
  // 如果有子路由
  if (route.children) {
    // Warn if route is named, does not redirect and has a default child route.
    // If users navigate to this route by name, the default child will
    // not be rendered (GH Issue #629)
    if (process.env.NODE_ENV !== 'production') {
      if (route.name && !route.redirect && route.children.some(child => /^\/?$/.test(child.path))) {
        warn(
          false,
          `Named Route '${route.name}' has a default child route. ` +
          `When navigating to this named route (:to="{name: '${route.name}'"), ` +
          `the default child route will not be rendered. Remove the name from ` +
          `this route and use the name of the default child route for named ` +
          `links instead.`
        )
      }
    }
    // 循环添加子路由路径
    route.children.forEach(child => {
      const childMatchAs = matchAs
        ? cleanPath(`${matchAs}/${child.path}`)
        : undefined
      addRouteRecord(pathList, pathMap, nameMap, child, record, childMatchAs)
    })
  }
  // 设置了路由别名
  if (route.alias !== undefined) {
    // 统一转换为数组
    const aliases = Array.isArray(route.alias)
      ? route.alias
      : [route.alias]

    aliases.forEach(alias => {
      const aliasRoute = {
        path: alias,
        children: route.children
      }
      addRouteRecord(
        pathList,
        pathMap,
        nameMap,
        aliasRoute,
        parent,
        record.path || '/' // matchAs
      )
    })
  }
  // 添加path记录以及建立path和记录的对应关系
  if (!pathMap[record.path]) {
    pathList.push(record.path)
    pathMap[record.path] = record
  }
  // 如果配置了命名路由，给name和record建立映射关系
  if (name) {
    if (!nameMap[name]) {
      nameMap[name] = record
    } else if (process.env.NODE_ENV !== 'production' && !matchAs) {
      // 已经有这个命名的路由存在并且没有给这个路由设置重定向，说明给了重复命名
      warn(
        false,
        `Duplicate named routes definition: ` +
        `{ name: "${name}", path: "${record.path}" }`
      )
    }
  }
}

function compileRouteRegex (path: string, pathToRegexpOptions: PathToRegexpOptions): RouteRegExp {
  const regex = Regexp(path, [], pathToRegexpOptions)
  if (process.env.NODE_ENV !== 'production') {
    const keys: any = Object.create(null)
    regex.keys.forEach(key => {
      // 有重复的动态路径参数
      warn(!keys[key.name], `Duplicate param keys in route with path: "${path}"`)
      keys[key.name] = true
    })
  }
  return regex
}

function normalizePath (path: string, parent?: RouteRecord, strict?: boolean): string {
  // 替换根路由路径
  if (!strict) path = path.replace(/\/$/, '')
  if (path[0] === '/') return path
  if (parent == null) return path
  // 将//的路径替换成/
  return cleanPath(`${parent.path}/${path}`)
}

```
这个方法的作用是将路由配置转换成一组组映射关系表，返回一个对象:

```javascript
  return {
    pathList,
    pathMap,
    nameMap
  }
```
其中`pathList`存储了所有的`path`，`pathMap`表示了`path`到`RouteRecord`对象的一一映射关系，`nameMap`则表示了`name`到`RouteRecord`对象的映射关系，`RouteRecord`对象是对路由配置参数`routes`每一项进行遍历后调用`addRouteRecord`方法生成的一条记录，方法定义如下：

```javascript
function addRouteRecord (
  pathList: Array<string>,
  pathMap: Dictionary<RouteRecord>,
  nameMap: Dictionary<RouteRecord>,
  route: RouteConfig,
  parent?: RouteRecord,
  matchAs?: string
) {
  const { path, name } = route
  if (process.env.NODE_ENV !== 'production') {
    // path不能为空并且component的值必须用一个组件名而不是一个string字符串
    assert(path != null, `"path" is required in a route configuration.`)
    assert(
      typeof route.component !== 'string',
      `route config "component" for path: ${String(path || name)} cannot be a ` +
      `string id. Use an actual component instead.`
    )
  }
  // pathToRegexpOptions是编译正则的选项
  const pathToRegexpOptions: PathToRegexpOptions = route.pathToRegexpOptions || {}
  // normalize path
  const normalizedPath = normalizePath(
    path,
    parent,
    pathToRegexpOptions.strict
  )
  // caseSensitive匹配规则是否大小写敏感？(默认值：false)
  if (typeof route.caseSensitive === 'boolean') {
    pathToRegexpOptions.sensitive = route.caseSensitive
  }

  const record: RouteRecord = {
    path: normalizedPath,
    regex: compileRouteRegex(normalizedPath, pathToRegexpOptions),
    components: route.components || { default: route.component },
    instances: {},
    name,
    parent,
    matchAs,
    redirect: route.redirect,
    beforeEnter: route.beforeEnter,
    meta: route.meta || {},
    props: route.props == null
      ? {}
      : route.components
        ? route.props
        : { default: route.props }
  }
  // 如果有子路由
  if (route.children) {
    // Warn if route is named, does not redirect and has a default child route.
    // If users navigate to this route by name, the default child will
    // not be rendered (GH Issue #629)
    if (process.env.NODE_ENV !== 'production') {
      if (route.name && !route.redirect && route.children.some(child => /^\/?$/.test(child.path))) {
        warn(
          false,
          `Named Route '${route.name}' has a default child route. ` +
          `When navigating to this named route (:to="{name: '${route.name}'"), ` +
          `the default child route will not be rendered. Remove the name from ` +
          `this route and use the name of the default child route for named ` +
          `links instead.`
        )
      }
    }
    // 循环添加子路由路径
    route.children.forEach(child => {
      const childMatchAs = matchAs
        ? cleanPath(`${matchAs}/${child.path}`)
        : undefined
      addRouteRecord(pathList, pathMap, nameMap, child, record, childMatchAs)
    })
  }
  // 设置了路由别名
  if (route.alias !== undefined) {
    // 统一转换为数组
    const aliases = Array.isArray(route.alias)
      ? route.alias
      : [route.alias]

    aliases.forEach(alias => {
      const aliasRoute = {
        path: alias,
        children: route.children
      }
      addRouteRecord(
        pathList,
        pathMap,
        nameMap,
        aliasRoute,
        parent,
        record.path || '/' // matchAs
      )
    })
  }
  // 添加path记录以及建立path和记录的对应关系
  if (!pathMap[record.path]) {
    pathList.push(record.path)
    pathMap[record.path] = record
  }
  // 如果配置了命名路由，给name和record建立映射关系
  if (name) {
    if (!nameMap[name]) {
      nameMap[name] = record
    } else if (process.env.NODE_ENV !== 'production' && !matchAs) {
      // 已经有这个命名的路由存在并且没有给这个路由设置重定向，说明给了重复命名
      warn(
        false,
        `Duplicate named routes definition: ` +
        `{ name: "${name}", path: "${record.path}" }`
      )
    }
  }
}
```
这个方法首先会对`path`使用`normalizePath`进行规范化处理，我们看一下这个方法：

```javascript
function normalizePath (path: string, parent?: RouteRecord, strict?: boolean): string {
  // 替换根路由路径
  if (!strict) path = path.replace(/\/$/, '')
  // 说明是一级路径，直接返回
  if (path[0] === '/') return path
  if (parent == null) return path
  //  将//的路径替换成/并且拼接父路由的路径
  return cleanPath(`${parent.path}/${path}`)
}
```
主要作用就是生成多层路由的具体路径，然后根据已有参数构建`RouteRecord`：

```
  const record: RouteRecord = {
    path: normalizedPath,
    regex: compileRouteRegex(normalizedPath, pathToRegexpOptions),
    components: route.components || { default: route.component },
    instances: {},
    name,
    parent,
    matchAs,
    redirect: route.redirect,
    beforeEnter: route.beforeEnter,
    meta: route.meta || {},
    props: route.props == null
      ? {}
      : route.components
        ? route.props
        : { default: route.props }
  }
```
这里解释下`regex`这个参数，用到了`path-to-regexp`这个库，这个库可以把路径转换为正则表达式，举个栗子：

```javascript
const keys = []
const regexp = pathToRegexp('/foo/:bar', keys)
// regexp = /^\/foo\/([^\/]+?)\/?$/i
// keys = [{ name: 'bar', prefix: '/', delimiter: '/', optional: false, repeat: false, pattern: '[^\\/]+?' }]

```
用这个库作为路径匹配引擎是为了实现可选的动态路径参数、匹配零个或多个、一个或多个，甚至是自定义正则匹配。
紧接着判断是否配置了子路由，然后循环调用`addRouteRecord`这个方法，并把当前的`record`作为`parent`，如果设置了路由别名，也会给别名添加一份`record`，最后就是更新映射表，返回一个`Array`对象以及两个`Dictionary`对象。

回到`create-matcher.js`，它对外暴露了两个方法：`addRoutes`和`match`，分别用于动态添加路由配置以及返回一个路由的路径，先看`addRoutes`：

```javascripts
function addRoutes (routes) {
    createRouteMap(routes, pathList, pathMap, nameMap)
  }
```
其实就是在现有的`pathList`、`pathMap`、`nameMap`上动态添加一条新纪录，这几个都是引用类型，执行`addRoutes`之后都会被修改。

`match`函数相对复杂一点，接受三个参数，第一个参数可以为`string`也可以是一个`Location`对象，第二个参数表示当前的路由路径，第三个参数也是`Location`对象，跟重定向有关：

```javascript

  function match (
    raw: RawLocation,
    currentRoute?: Route,
    redirectedFrom?: Location
  ): Route {
    // 根据raw和currentRoute计算出新的location
    const location = normalizeLocation(raw, currentRoute, false, router)
    const { name } = location

    if (name) {
      // 如果是命名路由，取出对应的路由record
      const record = nameMap[name]
      if (process.env.NODE_ENV !== 'production') {
        warn(record, `Route with name '${name}' does not exist`)
      }
      // 生成一条新记录
      if (!record) return _createRoute(null, location)
      const paramNames = record.regex.keys
        .filter(key => !key.optional)
        .map(key => key.name)

      if (typeof location.params !== 'object') {
        location.params = {}
      }
      // 赋值params
      if (currentRoute && typeof currentRoute.params === 'object') {
        for (const key in currentRoute.params) {
          if (!(key in location.params) && paramNames.indexOf(key) > -1) {
            location.params[key] = currentRoute.params[key]
          }
        }
      }

      if (record) {
        location.path = fillParams(record.path, location.params, `named route "${name}"`)
        return _createRoute(record, location, redirectedFrom)
      }
    } else if (location.path) {
      location.params = {}
      for (let i = 0; i < pathList.length; i++) {
        const path = pathList[i]
        const record = pathMap[path]
        if (matchRoute(record.regex, location.path, location.params)) {
          return _createRoute(record, location, redirectedFrom)
        }
      }
    }
    // no match
    return _createRoute(null, location)
  }
```
一开始会执行`normalizeLocation`方法，返回一个新的`location`，看一眼`normalizeLocation`的实现：

```javascript
export function normalizeLocation (
  raw: RawLocation,
  current: ?Route,
  append: ?boolean,
  router: ?VueRouter
): Location {
  let next: Location = typeof raw === 'string' ? { path: raw } : raw
  // named target
  if (next._normalized) {
    // 已经normalized的直接返回
    return next
  } else if (next.name) {
    // 如果是命名路由，返回一份备份
    return extend({}, raw)
  }

  // relative params
  // 没有path，但是有params和current
  if (!next.path && next.params && current) {
    next = extend({}, next)
    next._normalized = true
    // 拿到params
    const params: any = extend(extend({}, current.params), next.params)
    if (current.name) {
      next.name = current.name
      next.params = params
    } else if (current.matched.length) {
      const rawPath = current.matched[current.matched.length - 1].path
      // 根据rawPath和params计算出当前path
      next.path = fillParams(rawPath, params, `path ${current.path}`)
    } else if (process.env.NODE_ENV !== 'production') {
      warn(false, `relative params navigation requires a current route.`)
    }
    return next
  }
  // 将path拆分成path、hash和query
  const parsedPath = parsePath(next.path || '')
  const basePath = (current && current.path) || '/'
  // 返回最后拼接完成好的路径，append用于判断是否在当前 (相对) 路径前添加基路径
  const path = parsedPath.path
    ? resolvePath(parsedPath.path, basePath, append || next.append)
    : basePath
  // 解析query parseQuery是提供自定义查询字符串的解析/反解析函数，用于覆盖默认行为
  const query = resolveQuery(
    parsedPath.query,
    next.query,
    router && router.options.parseQuery
  )
  // 路由的hash值
  let hash = next.hash || parsedPath.hash
  if (hash && hash.charAt(0) !== '#') {
    hash = `#${hash}`
  }

  return {
    _normalized: true,
    path,
    query,
    hash
  }
}

```
这个方法首先会判断当前的`RawLocation`是否已经经过`_normalized`处理，是的话直接返回，否则的话继续判断当前`Location`是否有`name`字段，有的话通过`extend`方法拷贝一份`raw`对象直接返回，这个`extend`方法的实现很简单：

```javascript
export function extend (a, b) {
  for (const key in b) {
    a[key] = b[key]
  }
  return a
}
```
当上面的情况都不满足，接着进入下一个判断条件，如果有当前`Route`信息，有`params`但是没有`path`的情况，首先会设置`_normalized`标志位，然后对`params`参数进行合并处理，然后继续分为两种情况处理，分别是`current`有`name`与否，前者的话会直接将`current`的`name`和拼接后的`params`赋值给`next`后直接返回，后者的话会从路由记录里面找到最新的一条记录的`path`，调用`fillParams`方法根据`rawPath`和`params`计算当前`path`，看一下`fillParams`对相对路径的处理：

```javascript
/* @flow */

import { warn } from './warn'
import Regexp from 'path-to-regexp'

// $flow-disable-line
const regexpCompileCache: {
  [key: string]: Function
} = Object.create(null)

export function fillParams (
  path: string,
  params: ?Object,
  routeMsg: string
): string {
  params = params || {}
  try {
    const filler =
      regexpCompileCache[path] ||
      (regexpCompileCache[path] = Regexp.compile(path))
    // 如果param中有名为pathMatch的key将他设置为{0, params[patchMatch]}的键值对
    // Fix #2505 resolving asterisk routes { name: 'not-found', params: { pathMatch: '/not-found' }}
    if (params.pathMatch) params[0] = params.pathMatch
    // 将动态路径参数替换成正式参数
    return filler(params, { pretty: true })
  } catch (e) {
    if (process.env.NODE_ENV !== 'production') {
      warn(false, `missing param for ${routeMsg}: ${e.message}`)
    }
    return ''
  } finally {
    // delete the 0 if it was added
    delete params[0]
  }
}

```
其实这里就是把形如`{zapId: 1}`的params参数通过`Regexp.compile`生成的方法拼接到`path`后面，就像这样：

```javascript
const toPath = pathToRegexp.compile('/user/:id')

toPath({ id: 123 }) //=> "/user/123"
```
回到`create-matcher.js`，计算出新的`location`之后对命名路由和非命名路由进行了不同的处理，如果`name`存在，从`nameMap`字典里面匹配出对应的`record`，如果`record`不存在通过`_createRoute`生成一条新的`record`直接返回，否则取出这条`record`里面的`params`的`key`组成的数组，将`currentRoute`里面的`params`以`key`/`value`的形式不重复地存入名为`location.params`的一个对象里面，最后依然是通过`fillParams`拼接`params`参数到路径尾部，通过`_createRoute`方法创建一条新路径返回。

反之，如果是非命名路由，会通过`pathList`返回`path`对应的`record`，然后通过`createRoute`方法判断是否能够匹配到路由信息，是的话也会通过`_createRoute`生成一条新路径返回。接下来只要搞懂`matchRoute`和`_createRoute`干了什么就行了，先看`matchRoute`：

```javascript
function matchRoute (
  regex: RouteRegExp,
  path: string,
  params: Object
): boolean {
  const m = path.match(regex)

  if (!m) {
    return false
  } else if (!params) {
    return true
  }

  for (let i = 1, len = m.length; i < len; ++i) {
    const key = regex.keys[i - 1]
    const val = typeof m[i] === 'string' ? decodeURIComponent(m[i]) : m[i]
    if (key) {
      // Fix #1994: using * with props: true generates a param named 0
      params[key.name || 'pathMatch'] = val
    }
  }

  return true
}
```
其实就是通过`match`方法做判断，如果没匹配到直接返回`false`，如果传入了`params`会将`path`里面的`params`以`key`/`value`形式存入，这个传入的`params`在这里是`location.params`所以和前面做的是一样的操作。

接着看`_createRoute`方法，这个方法也在`create_matcher.js`内部：

```javascript

  function _createRoute (
    record: ?RouteRecord,
    location: Location,
    redirectedFrom?: Location
  ): Route {
    if (record && record.redirect) {
      return redirect(record, redirectedFrom || location)
    }
    if (record && record.matchAs) {
      return alias(record, location, record.matchAs)
    }
    return createRoute(record, location, redirectedFrom, router)
  }
```
无论是否设置了`redirect`还是`alias`最后都会重新调用`_createRoute`，所以这里直接看最后的`createRoute`方法：

```javascript
/* @flow */

import type VueRouter from '../index'
import { stringifyQuery } from './query'

const trailingSlashRE = /\/?$/

export function createRoute (
  record: ?RouteRecord,
  location: Location,
  redirectedFrom?: ?Location,
  router?: VueRouter
): Route {
  // 提供自定义查询字符串的解析/反解析函数。覆盖默认行为
  const stringifyQuery = router && router.options.stringifyQuery

  let query: any = location.query || {}
  try {
    query = clone(query)
  } catch (e) {}

  const route: Route = {
    name: location.name || (record && record.name),
    meta: (record && record.meta) || {},
    path: location.path || '/',
    hash: location.hash || '',
    query,
    params: location.params || {},
    fullPath: getFullPath(location, stringifyQuery), // 完整路径
    matched: record ? formatMatch(record) : []
  }
  if (redirectedFrom) {
    route.redirectedFrom = getFullPath(redirectedFrom, stringifyQuery)
  }
  return Object.freeze(route)
}

function clone (value) {
  if (Array.isArray(value)) {
    return value.map(clone)
  } else if (value && typeof value === 'object') {
    const res = {}
    for (const key in value) {
      res[key] = clone(value[key])
    }
    return res
  } else {
    return value
  }
}

// the starting route that represents the initial state
export const START = createRoute(null, {
  path: '/'
})

function formatMatch (record: ?RouteRecord): Array<RouteRecord> {
  const res = []
  while (record) {
    res.unshift(record)
    record = record.parent
  }
  return res
}

function getFullPath (
  { path, query = {}, hash = '' },
  _stringifyQuery
): string {
  const stringify = _stringifyQuery || stringifyQuery
  return (path || '/') + stringify(query) + hash
}

export function isSameRoute (a: Route, b: ?Route): boolean {
  if (b === START) {
    return a === b
  } else if (!b) {
    return false
  } else if (a.path && b.path) {
    return (
      a.path.replace(trailingSlashRE, '') === b.path.replace(trailingSlashRE, '') &&
      a.hash === b.hash &&
      isObjectEqual(a.query, b.query)
    )
  } else if (a.name && b.name) {
    return (
      a.name === b.name &&
      a.hash === b.hash &&
      isObjectEqual(a.query, b.query) &&
      isObjectEqual(a.params, b.params)
    )
  } else {
    return false
  }
}

function isObjectEqual (a = {}, b = {}): boolean {
  // handle null value #1566
  if (!a || !b) return a === b
  const aKeys = Object.keys(a)
  const bKeys = Object.keys(b)
  if (aKeys.length !== bKeys.length) {
    return false
  }
  return aKeys.every(key => {
    const aVal = a[key]
    const bVal = b[key]
    // check nested equality
    if (typeof aVal === 'object' && typeof bVal === 'object') {
      return isObjectEqual(aVal, bVal)
    }
    return String(aVal) === String(bVal)
  })
}

export function isIncludedRoute (current: Route, target: Route): boolean {
  return (
    current.path.replace(trailingSlashRE, '/').indexOf(
      target.path.replace(trailingSlashRE, '/')
    ) === 0 &&
    (!target.hash || current.hash === target.hash) &&
    queryIncludes(current.query, target.query)
  )
}

function queryIncludes (current: Dictionary<string>, target: Dictionary<string>): boolean {
  for (const key in target) {
    if (!(key in current)) {
      return false
    }
  }
  return true
}

```
通过传入的`record`和`location`创建一个不可被修改的`Route`对象，其中有个`matched`属性通过`formatMatch`方法构建：

```javascript
function formatMatch (record: ?RouteRecord): Array<RouteRecord> {
  const res = []
  while (record) {
    res.unshift(record)
    record = record.parent
  }
  return res
}
```
通过循环不断地查找当前`record`的`parent`，然后返回这条线上所有的`record`组成的数组。

##### 路由的跳转
无论是`Hash`路由还是`History`路由，在初始化的时候都会通过`transitionTo`方法跳转到初始路径，这个方法也是我们切换路由路径时候使用的方法，下面分析一下该方法的实现过程：

```javascript
 transitionTo (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    // 通过location和current返回初始化Route信息
    const route = this.router.match(location, this.current)
    this.confirmTransition(route, () => {
      this.updateRoute(route)
      // 执行onComplete回调
      onComplete && onComplete(route)
      this.ensureURL()

      // fire ready cbs once
      if (!this.ready) {
        this.ready = true
        this.readyCbs.forEach(cb => { cb(route) })
      }
    }, err => {
      // 如果跳转被终止
      if (onAbort) {
        onAbort(err)
      }
      if (err && !this.ready) {
        this.ready = true
        this.readyErrorCbs.forEach(cb => { cb(err) })
      }
    })
  }
```
初始化调用的时候，拿到的是初始化的`Route`，通过`confirmTransition`方法进行实际的跳转操作，同时对跳转成功和失败两种情况都设置了回调函数，看一下这个函数的定义：

```javascript

  confirmTransition (route: Route, onComplete: Function, onAbort?: Function) {
    const current = this.current
    const abort = err => {
      if (isError(err)) {
        // 将所有的错误信息都存入errorCbs
        if (this.errorCbs.length) {
          this.errorCbs.forEach(cb => { cb(err) })
        } else {
          warn(false, 'uncaught error during route navigation:')
          console.error(err)
        }
      }
      onAbort && onAbort(err)
    }
    if (
      // 如果是同一个路由路径
      isSameRoute(route, current) &&
      // in the case the route map has been dynamically appended to
      route.matched.length === current.matched.length
    ) {
      // ensureURL在不同的路由实现方式里面该方法的实现不一样
      this.ensureURL()
      // 如果设置了abort方法这里直接调用
      return abort()
    }
    // 拿到路径的变化部分以及遗弃部分和升级部分
    const {
      updated,
      deactivated,
      activated
    } = resolveQueue(this.current.matched, route.matched)
    // 维持一个对应路径变化的导航守卫的钩子组成的List
    const queue: Array<?NavigationGuard> = [].concat(
      // in-component leave guards
      extractLeaveGuards(deactivated),
      // global before hooks
      this.router.beforeHooks,
      // in-component update hooks
      extractUpdateHooks(updated),
      // in-config enter guards
      activated.map(m => m.beforeEnter),
      // async components
      resolveAsyncComponents(activated)
    )

    this.pending = route
    // 定义一个迭代器
    const iterator = (hook: NavigationGuard, next) => {
      if (this.pending !== route) {
        return abort()
      }
      try {
        hook(route, current, (to: any) => {
          if (to === false || isError(to)) {
            // next(false) -> abort navigation, ensure current URL
            this.ensureURL(true)
            abort(to)
          } else if (
            typeof to === 'string' ||
            (typeof to === 'object' && (
              typeof to.path === 'string' ||
              typeof to.name === 'string'
            ))
          ) {
            // next('/') or next({ path: '/' }) -> redirect
            abort()
            if (typeof to === 'object' && to.replace) {
              this.replace(to)
            } else {
              this.push(to)
            }
          } else {
            // 确认跳转，执行回调
            // confirm transition and pass on the value
            next(to)
          }
        })
      } catch (e) {
        abort(e)
      }
    }

    runQueue(queue, iterator, () => {
      const postEnterCbs = []
      const isValid = () => this.current === route
      // wait until async components are resolved before
      // extracting in-component enter guards
      // 执行beforeRouteEnter钩子函数
      const enterGuards = extractEnterGuards(activated, postEnterCbs, isValid)
      const queue = enterGuards.concat(this.router.resolveHooks)
      runQueue(queue, iterator, () => {
        if (this.pending !== route) {
          return abort()
        }
        this.pending = null
        onComplete(route)
        if (this.router.app) {
          this.router.app.$nextTick(() => {
            postEnterCbs.forEach(cb => { cb() })
          })
        }
      })
    })
  }
```
首先定义了一个`abort`函数用于处理路由跳转失败的情况以及执行`onAbort`回调，然后判断如果要跳转的路由和当前路由是同一个的话，直接调用`this.ensureURL()`和`abort()`，这个`ensureURL`在不同的路由实现方式里面该方法的实现不一样，最终都是做了路由跳转的操作，紧接着通过`resolveQueue`方法拿到三个数组，分别存储着固定的部分，遗弃的部分以及更新的部分，这个方法的实现如下：

```javascript
function resolveQueue (
  current: Array<RouteRecord>,
  next: Array<RouteRecord>
): {
  updated: Array<RouteRecord>,
  activated: Array<RouteRecord>,
  deactivated: Array<RouteRecord>
} {
  let i
  const max = Math.max(current.length, next.length)
  for (i = 0; i < max; i++) {
    if (current[i] !== next[i]) {
      break
    }
  }
  return {
    updated: next.slice(0, i),
    activated: next.slice(i),
    deactivated: current.slice(i)
  }
}
```
路由从`current`变为`next`，两个路径的公共部分就是`next.slice(0, i)`，`next.slice(0, i)`就是路径需要更新的部分，而`current.slice(i)`就是路径需要变化的部分。

拿到三个数组之后会构造一个`queue`队列，里面存储了路径变化要执行的钩子函数，也就是官方说的路由守卫，会按次序执行一些诸如`beforeRouteLeave`、`beforeRouteUpdate `等方法，下面还会定义一个迭代器，这个迭代器会根据传入的`ro`参数来决定执行`abort`还是`next`方法，最后执行`runQueue`方法来执行这个队列，这个方法的定义如下：

```javascript
export function runQueue (queue: Array<?NavigationGuard>, fn: Function, cb: Function) {
  const step = index => {
    if (index >= queue.length) {
      cb()
    } else {
      if (queue[index]) {
        fn(queue[index], () => {
          step(index + 1)
        })
      } else {
        step(index + 1)
      }
    }
  }
  step(0)
}
```
这里的`fn`其实就是`iterator`里面的`next`函数，只有执行了`next`函数`index`才会`+1`，才会进行管道中的下一个钩子，如果全部钩子执行完了，则导航的状态会变成`confirmed`(确认的)。

最后可以看一下`Vue-Router`里面对导航的完整解析流程：

> 导航被触发。
在失活的组件里调用离开守卫。
调用全局的 beforeEach 守卫。
在重用的组件里调用 beforeRouteUpdate 守卫 (2.2+)。
在路由配置里调用 beforeEnter。
解析异步路由组件。
在被激活的组件里调用 beforeRouteEnter。
调用全局的 beforeResolve 守卫 (2.5+)。
导航被确认。
调用全局的 afterEach 钩子。
触发 DOM 更新。
用创建好的实例调用 beforeRouteEnter 守卫中传给 next 的回调函数。

#### 总结
传统的路由实现是通过对路径的切换做到的，对于`Vue-Router`而言，路由模块的本质 就是建立起url和页面之间的映射关系。路由始终会维护当前的线路，路由切换的时候会把当前线路切换到目标线路，切换过程中会执行一系列的导航守卫钩子函数，会更改`url`，同样也会渲染对应的组件，切换完毕后会把目标线路更新替换当前线路，这样就会作为下一次的路径切换的依据.

参考链接：[Vue核心解密](https://ustbhuangyi.github.io/vue-analysis/vue-router/)
