redux作为大型应用的状态管理工具，如果想配合react使用，需要借助react-redux。 redux主要完成两件事情：
- 通过`context`从`root`向下传入`store`，保证数据的单项流动的同时也方便了子组件从`store`上获取数据
- 当应用状态发生变化，触发`subscribe`方法进行监听，实现相关逻辑

> 在`React 16.4.0`之前，`React`官方是不推荐使用`context`的，原因在于，当`context`中的值刷新的时候，是从上到下刷新的，如果中间有组件的`shouldComponentUpdate`返回了`false`，这个组件下面的组件就收不到更新后的值；而`React-Redux`实现了订阅发布的模式，保证使用了`store`的组件在数据更新的时候可以得到通知。
在`React 16.4.0`之后官方将`createContext`暴露出来了，以上的问题不会出现，但是是不是意味着，可以用`context`来替代`redux`呢？理论上是可以的，但是并不推荐这样做，因为在`redux`的发展中，其生态系统是非常繁荣的，用`Redux`能避免重复造轮子的窘境。 
引自：http://cuteshilina.com/2019/01/19/HowReactReduxWorks/#我们为什么需要react-redux

看一眼React-Redux的目录结构：

```javascript
├── components
│   ├── Context.js
│   ├── Provider.js
│   └── connectAdvanced.js
├── connect
│   ├── connect.js
│   ├── mapDispatchToProps.js
│   ├── mapStateToProps.js
│   ├── mergeProps.js
│   ├── selectorFactory.js
│   ├── verifySubselectors.js
│   └── wrapMapToProps.js
├── index.js
└── utils
    ├── isPlainObject.js
    ├── shallowEqual.js
    ├── verifyPlainObject.js
```
入口在`index.js`：

#### Index
```javascript
import Provider from './components/Provider'
import connectAdvanced from './components/connectAdvanced'
import { ReactReduxContext } from './components/Context'
import connect from './connect/connect'

export { Provider, connectAdvanced, ReactReduxContext, connect }
```
可以看到入口只是对外输出了`Provider`、`connenct`、`connectedAdvanced`三个方法，我们常用的是前面两种方法。

先看Provider。

#### Provider
```javascript
import React, { Component } from 'react'
import PropTypes from 'prop-types'
import { ReactReduxContext } from './Context'

class Provider extends Component {
  constructor(props) {
    super(props)
    // 获取store
    const { store } = props

    this.state = {
      storeState: store.getState(),
      store
    }
  }

  componentDidMount() {
    // 判断是否挂载的flag
    this._isMounted = true
    this.subscribe()
  }

  componentWillUnmount() {
    if (this.unsubscribe) this.unsubscribe()

    this._isMounted = false
  }

  componentDidUpdate(prevProps) {
    if (this.props.store !== prevProps.store) {
      if (this.unsubscribe) this.unsubscribe()

      this.subscribe()
    }
  }

  subscribe() {
    const { store } = this.props

    this.unsubscribe = store.subscribe(() => {
      // 获取最新的state
      const newStoreState = store.getState()

      if (!this._isMounted) {
        return
      }
      // 更新storeState
      this.setState(providerState => {
        // If the value is the same, skip the unnecessary state update.
        if (providerState.storeState === newStoreState) {
          return null
        }

        return { storeState: newStoreState }
      })
    })

    // Actions might have been dispatched between render and mount - handle those
    const postMountStoreState = store.getState()
    if (postMountStoreState !== this.state.storeState) {
      this.setState({ storeState: postMountStoreState })
    }
  }

  render() {
    const Context = this.props.context || ReactReduxContext

    return (
      <Context.Provider value={this.state}>
        {this.props.children}
      </Context.Provider>
    )
  }
}

Provider.propTypes = {
  store: PropTypes.shape({
    subscribe: PropTypes.func.isRequired,
    dispatch: PropTypes.func.isRequired,
    getState: PropTypes.func.isRequired
  }),
  context: PropTypes.object,
  children: PropTypes.any
}

export default Provider
```
我们在使用Provider的时候只需要把`Store`传入，通过`Context`这个`API`来实现全局注入Store，除了使用默认的`Context`之外我们也可以使用自定义的`Context`，看一下默认的实现：
```javascript
import React from 'react'

export const ReactReduxContext = React.createContext(null)

export default ReactReduxContext
```
回到`Provider.js`，在初始化阶段利用`state`去存储`store`及其对应的`state`信息，在`componentDidMount`周期调用`subscribe`方法添加订阅以便于在每次`dispatch action`的时候可以更新当前`state`里面的`storeState`，同时这个方法会返回一个用于取消当前订阅的函数，在`componentWillUnmount`时候执行。
可以看到`Provider`做的工作其实比较简单，事实上也是如此，主要的工作在`connect.js`。

#### Connect

```javascript
import connectAdvanced from '../components/connectAdvanced'
import shallowEqual from '../utils/shallowEqual'
import defaultMapDispatchToPropsFactories from './mapDispatchToProps'
import defaultMapStateToPropsFactories from './mapStateToProps'
import defaultMergePropsFactories from './mergeProps'
import defaultSelectorFactory from './selectorFactory'

/*
  connect is a facade over connectAdvanced. It turns its args into a compatible
  selectorFactory, which has the signature:

    (dispatch, options) => (nextState, nextOwnProps) => nextFinalProps

  connect passes its args to connectAdvanced as options, which will in turn pass them to
  selectorFactory each time a Connect component instance is instantiated or hot reloaded.

  selectorFactory returns a final props selector from its mapStateToProps,
  mapStateToPropsFactories, mapDispatchToProps, mapDispatchToPropsFactories, mergeProps,
  mergePropsFactories, and pure args.

  The resulting final props selector is called by the Connect component instance whenever
  it receives new props or store state.
 */

function match(arg, factories, name) {
  for (let i = factories.length - 1; i >= 0; i--) {
    const result = factories[i](arg)
    if (result) return result
  }

  return (dispatch, options) => {
    throw new Error(
      `Invalid value of type ${typeof arg} for ${name} argument when connecting component ${
        options.wrappedComponentName
      }.`
    )
  }
}

function strictEqual(a, b) {
  return a === b
}

// createConnect with default args builds the 'official' connect behavior. Calling it with
// different options opens up some testing and extensibility scenarios
export function createConnect({
  connectHOC = connectAdvanced,
  mapStateToPropsFactories = defaultMapStateToPropsFactories, // 根据传入的mapStateToProp参数类型来决定调用什么方法
  mapDispatchToPropsFactories = defaultMapDispatchToPropsFactories, // 根据传入的mapDispatchToProps参数类型来决定调用什么方法
  mergePropsFactories = defaultMergePropsFactories, // 如果不传调用默认的mergeProps方法，否则使用自定义的mergeProps方法
  selectorFactory = defaultSelectorFactory // 计算mapStateToProps，mapDispatchToProps, ownProps的结果
} = {}) {
  return function connect(
    mapStateToProps,
    mapDispatchToProps,
    mergeProps,
    {
      pure = true, // 设为true表示我们假设这个组件的状态除了从props传入之外不依赖于外界输入以及自身的state，此时connect只会对相关的state/props进行浅比较以避免重新渲染
      areStatesEqual = strictEqual,
      areOwnPropsEqual = shallowEqual,
      areStatePropsEqual = shallowEqual,
      areMergedPropsEqual = shallowEqual,
      ...extraOptions
    } = {}
  ) {
    // 根据传入的参数的类型不同Object/Function调用对应的factory方法来返回对应的函数
    const initMapStateToProps = match(
      mapStateToProps,
      mapStateToPropsFactories,
      'mapStateToProps'
    )
    const initMapDispatchToProps = match(
      mapDispatchToProps,
      mapDispatchToPropsFactories,
      'mapDispatchToProps'
    )
    const initMergeProps = match(mergeProps, mergePropsFactories, 'mergeProps')

    return connectHOC(selectorFactory, {
      // used in error messages
      methodName: 'connect',

      // used to compute Connect's displayName from the wrapped component's displayName.
      getDisplayName: name => `Connect(${name})`,

      // if mapStateToProps is falsy, the Connect component doesn't subscribe to store state changes
      // 如果没有传mapStateToProps将不会监听state的变化
      shouldHandleStateChanges: Boolean(mapStateToProps),

      // passed through to selectorFactory
      // 传递给selectFactory的参数
      initMapStateToProps,
      initMapDispatchToProps,
      initMergeProps,
      pure,
      areStatesEqual,
      areOwnPropsEqual,
      areStatePropsEqual,
      areMergedPropsEqual,

      // any extra options args can override defaults of connect or connectAdvanced
      ...extraOptions
    })
  }
}

export default createConnect()

```
要看懂`connect.js`需要结合`connectAdvanced.js`一起，因为后者是前者的基础，这个函数会将`React`组件和`Redux Store`链接在一起，但是这个方法不会去处理如何将`state`、`ownProps`等组合到`props`里面，决定其行为的还是`connect.js`。

通过`export default createConnect()`可以知道`connect.js`对外输出的是我们平常使用的`connect`方法，这个方法接受`mapStateToProps`、`mapDispatchToProps`、`mergeProps`等主要参数以及一些可选参数。

紧接着，通过`match`方法，根据我们传入的参数类型不同，调用不同的`xxfactory`方法，以`mapStateToProps`为例，我们可以选择传入一个函数或者不传，会得到不同的调用结果：

```javascript
import { wrapMapToPropsConstant, wrapMapToPropsFunc } from './wrapMapToProps'

export function whenMapStateToPropsIsFunction(mapStateToProps) {
  return typeof mapStateToProps === 'function'
    ? wrapMapToPropsFunc(mapStateToProps, 'mapStateToProps')
    : undefined
}

export function whenMapStateToPropsIsMissing(mapStateToProps) {
  return !mapStateToProps ? wrapMapToPropsConstant(() => ({})) : undefined
}

export default [whenMapStateToPropsIsFunction, whenMapStateToPropsIsMissing]

```
如果不传`mapStateToProps`会调用`wrapMapToPropsConstant`方法，该方法以一个返回空对象的函数作为参数，最后其实还是返回一个空对象：

```javascript
export function wrapMapToPropsConstant(getConstant) {
  return function initConstantSelector(dispatch, options) {
    const constant = getConstant(dispatch, options)

    function constantSelector() {
      return constant
    }
    constantSelector.dependsOnOwnProps = false
    return constantSelector
  }
}
```
否则的话就调用`wrapMapToPropsFunc`方法：

```javascript
export function wrapMapToPropsFunc(mapToProps, methodName) {
  return function initProxySelector(dispatch, { displayName }) {
    const proxy = function mapToPropsProxy(stateOrDispatch, ownProps) {
      return proxy.dependsOnOwnProps
        ? proxy.mapToProps(stateOrDispatch, ownProps)
        : proxy.mapToProps(stateOrDispatch)
    }

    // allow detectFactoryAndVerify to get ownProps
    proxy.dependsOnOwnProps = true

    proxy.mapToProps = function detectFactoryAndVerify(
      stateOrDispatch,
      ownProps
    ) {
      proxy.mapToProps = mapToProps
      // 检查是否订阅了ownProps
      proxy.dependsOnOwnProps = getDependsOnOwnProps(mapToProps)
      let props = proxy(stateOrDispatch, ownProps)

      if (typeof props === 'function') {
        proxy.mapToProps = props
        proxy.dependsOnOwnProps = getDependsOnOwnProps(props)
        props = proxy(stateOrDispatch, ownProps)
      }

      if (process.env.NODE_ENV !== 'production')
        verifyPlainObject(props, displayName, methodName)

      return props
    }

    return proxy
  }
}
```
这里会返回一个名为`initProxySelector`的接受`dispatch`作为参数的函数，关于这个函数细节待会回来看，先回到`connect.js`，接着往下看。

在做完上述工作后，直接将所有参数传入`connectHOC`，所以我们要看一下`connectHOC`做了什么：

#### connectAdvanced

```javascript
import hoistStatics from 'hoist-non-React-statics'
import invariant from 'invariant'
import React, { Component, PureComponent } from 'react'
import { isValidElementType, isContextConsumer } from 'React-is'

import { ReactReduxContext } from './Context'

const stringifyComponent = Comp => {
  try {
    return JSON.stringify(Comp)
  } catch (err) {
    return String(Comp)
  }
}

export default function connectAdvanced(
  /*
    selectorFactory is a func that is responsible for returning the selector function used to
    compute new props from state, props, and dispatch. For example:

      export default connectAdvanced((dispatch, options) => (state, props) => ({
        thing: state.things[props.thingId],
        saveThing: fields => dispatch(actionCreators.saveThing(props.thingId, fields)),
      }))(YourComponent)

    Access to dispatch is provided to the factory so selectorFactories can bind actionCreators
    outside of their selector as an optimization. Options passed to connectAdvanced are passed to
    the selectorFactory, along with displayName and WrappedComponent, as the second argument.

    Note that selectorFactory is responsible for all caching/memoization of inbound and outbound
    props. Do not use connectAdvanced directly without memoizing results between calls to your
    selector, otherwise the Connect component will re-render on every state or props change.
  */
  selectorFactory,
  // options object:
  {
    // the func used to compute this HOC's displayName from the wrapped component's displayName.
    // probably overridden by wrapper functions such as connect()
    getDisplayName = name => `ConnectAdvanced(${name})`,

    // shown in error messages
    // probably overridden by wrapper functions such as connect()
    methodName = 'connectAdvanced',

    // REMOVED: if defined, the name of the property passed to the wrapped element indicating the number of
    // calls to render. useful for watching in React devtools for unnecessary re-renders.
    renderCountProp = undefined,

    // determines whether this HOC subscribes to store changes
    shouldHandleStateChanges = true,

    // REMOVED: the key of props/context to get the store
    storeKey = 'store',

    // REMOVED: expose the wrapped component via refs
    withRef = false,

    // use React's forwardRef to expose a ref of the wrapped component
    forwardRef = false,

    // the context consumer to use
    context = ReactReduxContext,

    // additional options are passed through to the selectorFactory
    ...connectOptions
  } = {}
) {
  invariant(
    renderCountProp === undefined,
    `renderCountProp is removed. render counting is built into the latest React dev tools profiling extension`
  )

  invariant(
    !withRef,
    'withRef is removed. To access the wrapped instance, use a ref on the connected component'
  )

  const customStoreWarningMessage =
    'To use a custom Redux store for specific components,  create a custom React context with ' +
    "React.createContext(), and pass the context object to React Redux's Provider and specific components" +
    ' like:  <Provider context={MyContext}><ConnectedComponent context={MyContext} /></Provider>. ' +
    'You may also pass a {context : MyContext} option to connect'

  invariant(
    storeKey === 'store',
    'storeKey has been removed and does not do anything. ' +
      customStoreWarningMessage
  )
  // 存储context
  const Context = context

  return function wrapWithConnect(WrappedComponent) {
    if (process.env.NODE_ENV !== 'production') {
      invariant(
        isValidElementType(WrappedComponent),
        `You must pass a component to the function returned by ` +
          `${methodName}. Instead received ${stringifyComponent(
            WrappedComponent
          )}`
      )
    }

    const wrappedComponentName =
      WrappedComponent.displayName || WrappedComponent.name || 'Component'

    const displayName = getDisplayName(wrappedComponentName)

    const selectorFactoryOptions = {
      ...connectOptions,
      getDisplayName,
      methodName,
      renderCountProp,
      shouldHandleStateChanges,
      storeKey,
      displayName,
      wrappedComponentName,
      WrappedComponent
    }

    const { pure } = connectOptions
    // pure参数决定是继承Component还是PureComponent
    let OuterBaseComponent = Component

    if (pure) {
      OuterBaseComponent = PureComponent
    }

    function makeDerivedPropsSelector() {
      let lastProps
      let lastState
      let lastDerivedProps
      let lastStore
      let lastSelectorFactoryOptions
      let sourceSelector

      return function selectDerivedProps(
        state,
        props,
        store,
        selectorFactoryOptions
      ) {
        if (pure && lastProps === props && lastState === state) {
          // 直接返回上一次生成的props，避免不必要的渲染工作
          return lastDerivedProps
        }

        if (
          store !== lastStore ||
          lastSelectorFactoryOptions !== selectorFactoryOptions
        ) {
          // 更新数据
          lastStore = store
          lastSelectorFactoryOptions = selectorFactoryOptions
          sourceSelector = selectorFactory(
            store.dispatch,
            selectorFactoryOptions
          )
        }

        lastProps = props
        lastState = state
        // 生成新的需要注入的props，这里传入的props是ownProps
        const nextProps = sourceSelector(state, props)

        lastDerivedProps = nextProps
        return lastDerivedProps
      }
    }

    function makeChildElementSelector() {
      let lastChildProps, lastForwardRef, lastChildElement, lastComponent

      return function selectChildElement(
        WrappedComponent,
        childProps,
        forwardRef
      ) {
        if (
          childProps !== lastChildProps ||
          forwardRef !== lastForwardRef ||
          lastComponent !== WrappedComponent
        ) {
          lastChildProps = childProps
          lastForwardRef = forwardRef
          lastComponent = WrappedComponent
          lastChildElement = (
            <WrappedComponent {...childProps} ref={forwardRef} />
          )
        }

        return lastChildElement
      }
    }

    class Connect extends OuterBaseComponent {
      constructor(props) {
        super(props)
        invariant(
          forwardRef ? !props.wrapperProps[storeKey] : !props[storeKey],
          'Passing redux store in props has been removed and does not do anything. ' +
            customStoreWarningMessage
        )
        this.selectDerivedProps = makeDerivedPropsSelector()
        this.selectChildElement = makeChildElementSelector()
        this.indirectRenderWrappedComponent = this.indirectRenderWrappedComponent.bind(
          this
        )
      }

      indirectRenderWrappedComponent(value) {
        // calling renderWrappedComponent on prototype from indirectRenderWrappedComponent bound to `this`
        return this.renderWrappedComponent(value)
      }

      renderWrappedComponent(value) {
        // 这里的value就是最开始在Provider里面定义的state
        invariant(
          value,
          `Could not find "store" in the context of ` +
            `"${displayName}". Either wrap the root component in a <Provider>, ` +
            `or pass a custom React context provider to <Provider> and the corresponding ` +
            `React context consumer to ${displayName} in connect options.`
        )
        const { storeState, store } = value

        let wrapperProps = this.props
        let forwardedRef

        if (forwardRef) {
          wrapperProps = this.props.wrapperProps
          forwardedRef = this.props.forwardedRef
        }
        // 生成要将store中哪些数据注入props的方法
        let derivedProps = this.selectDerivedProps(
          storeState,
          wrapperProps,
          store,
          selectorFactoryOptions
        )
        // 拼装最后需要返回给connectAdvanced的组件
        return this.selectChildElement(
          WrappedComponent,
          derivedProps,
          forwardedRef
        )
      }

      render() {
        // 获取context以便使用context.consumer获取store的变更
        const ContextToUse =
          this.props.context &&
          this.props.context.Consumer &&
          isContextConsumer(<this.props.context.Consumer />)
            ? this.props.context
            : Context

        return (
          <ContextToUse.Consumer>
            {this.indirectRenderWrappedComponent}
          </ContextToUse.Consumer>
        )
      }
    }

    Connect.WrappedComponent = WrappedComponent
    Connect.displayName = displayName

    if (forwardRef) {
      // 将ref转发给子组件
      const forwarded = React.forwardRef(function forwardConnectRef(
        props,
        ref
      ) {
        return <Connect wrapperProps={props} forwardedRef={ref} />
      })

      forwarded.displayName = displayName
      forwarded.WrappedComponent = WrappedComponent
      // hoistStatics用于copy静态方法，避免在使用HOC的时候类的静态方法丢失
      return hoistStatics(forwarded, WrappedComponent)
    }

    return hoistStatics(Connect, WrappedComponent)
  }
}

```
略过前面的一堆警告判断不看，直接看返回，`connectAdvanced`会返回一个叫`wrapWithConnect`的方法，这个方法以一个`React Component`作为参数，回想我们平常调用`connect`的时候：

```javascript
connect(mapStateToProps,mapDispatchToProps,mergeProps,options)(App);
```
就是在这里，这个函数会先获取到`wrappedComponentName`，然后将除了`connect`方法前三个参数之外的其他参数都包裹到了`selectorFactoryOptions`里面，`wrapWithConnect`这个`HOC`会返回一个新的组件给我们，至于新组件是继承`Component`还是`PureComponent`取决于我们配置的`pure`参数，这里跳过接下来的两个方法直接看`return`，因为用到了`HOC`这里有两个需要注意的问题，一个是`ref`转发，这里代码里面有相关体现，另一个就是在使用`HOC`的时候会造成`static`丢失的问题，这里通过`hoist-non-react-statics`进行了处理，具体的描述可以看[react doc](https://reactjs.org/docs/higher-order-components.html#static-methods-must-be-copied-over)。

`render`的时候会调用`this.indirectRenderWrappedComponent`方法，这个方法接收当前的`context`值，也就是`Provider`里面提供的`state`，看一眼这个方法：

```javascript
renderWrappedComponent(value) {
        // 这里的value就是最开始在Provider里面定义的state
        invariant(
          value,
          `Could not find "store" in the context of ` +
            `"${displayName}". Either wrap the root component in a <Provider>, ` +
            `or pass a custom React context provider to <Provider> and the corresponding ` +
            `React context consumer to ${displayName} in connect options.`
        )
        const { storeState, store } = value

        let wrapperProps = this.props
        let forwardedRef

        if (forwardRef) {
          wrapperProps = this.props.wrapperProps
          forwardedRef = this.props.forwardedRef
        }
        // 生成要将store中哪些数据注入props的方法
        let derivedProps = this.selectDerivedProps(
          storeState,
          wrapperProps,
          store,
          selectorFactoryOptions
        )
        // 拼装最后需要返回给connectAdvanced的组件
        return this.selectChildElement(
          WrappedComponent,
          derivedProps,
          forwardedRef
        )
      }
```
前面的不看，直接看`selectDerivedProps`，其实是调用了`makeDerivedPropsSelector`：

```javascript
function makeDerivedPropsSelector() {
      let lastProps
      let lastState
      let lastDerivedProps
      let lastStore
      let lastSelectorFactoryOptions
      let sourceSelector

      return function selectDerivedProps(
        state,
        props,
        store,
        selectorFactoryOptions
      ) {
        if (pure && lastProps === props && lastState === state) {
          // 直接返回上一次生成的props，避免不必要的渲染工作
          return lastDerivedProps
        }

        if (
          store !== lastStore ||
          lastSelectorFactoryOptions !== selectorFactoryOptions
        ) {
          // 更新数据
          lastStore = store
          lastSelectorFactoryOptions = selectorFactoryOptions
          sourceSelector = selectorFactory(
            store.dispatch,
            selectorFactoryOptions
          )
        }

        lastProps = props
        lastState = state
        // 生成新的需要注入的props，这里传入的props是ownProps
        const nextProps = sourceSelector(state, props)

        lastDerivedProps = nextProps
        return lastDerivedProps
      }
    }
```
首先会对`state`和`props`做一次浅比较，如果没有变化直接返回上一次计算得到的结果，否则将`dispatch`和`selectorFactoryOptions`传入`selectorFactory`得到`sourceSelector`，`sourceSelector`由`connect`传入，这个方法用于计算`mapStateToProps`、`mapDispatchToProps`、 `ownProps`的结果：

```javascript
export default function finalPropsSelectorFactory(
  dispatch,
  { initMapStateToProps, initMapDispatchToProps, initMergeProps, ...options }
) {
  const mapStateToProps = initMapStateToProps(dispatch, options)
  const mapDispatchToProps = initMapDispatchToProps(dispatch, options)
  const mergeProps = initMergeProps(dispatch, options)

  if (process.env.NODE_ENV !== 'production') {
    verifySubselectors(
      mapStateToProps,
      mapDispatchToProps,
      mergeProps,
      options.displayName
    )
  }

  const selectorFactory = options.pure
    ? pureFinalPropsSelectorFactory
    : impureFinalPropsSelectorFactory

  return selectorFactory(
    mapStateToProps,
    mapDispatchToProps,
    mergeProps,
    dispatch,
    options
  )
}
```
这里的`initMapStateToProps`之流就是我们之前`wrapMapToPropsFunc`函数返回的结果，我把代码贴下面方便对照：

```javascript
export function wrapMapToPropsFunc(mapToProps, methodName) {
  return function initProxySelector(dispatch, { displayName }) {
    const proxy = function mapToPropsProxy(stateOrDispatch, ownProps) {
      return proxy.dependsOnOwnProps
        ? proxy.mapToProps(stateOrDispatch, ownProps)
        : proxy.mapToProps(stateOrDispatch)
    }

    // allow detectFactoryAndVerify to get ownProps
    proxy.dependsOnOwnProps = true

    proxy.mapToProps = function detectFactoryAndVerify(
      stateOrDispatch,
      ownProps
    ) {
      proxy.mapToProps = mapToProps
      // 检查是否订阅了ownProps
      proxy.dependsOnOwnProps = getDependsOnOwnProps(mapToProps)
      let props = proxy(stateOrDispatch, ownProps)

      if (typeof props === 'function') {
        proxy.mapToProps = props
        proxy.dependsOnOwnProps = getDependsOnOwnProps(props)
        props = proxy(stateOrDispatch, ownProps)
      }

      if (process.env.NODE_ENV !== 'production')
        verifyPlainObject(props, displayName, methodName)

      return props
    }

    return proxy
  }
}
```
还是用`mapStateToProps`为例，先看`finalPropsSelectorFactory`，如果我们在参数里面设置了`pure`，就会调用`pureFinalPropsSelectorFactory`去构造最后的`selectorFactory`函数，这个函数定义如下：

```javascript

export function pureFinalPropsSelectorFactory(
  mapStateToProps,
  mapDispatchToProps,
  mergeProps,
  dispatch,
  { areStatesEqual, areOwnPropsEqual, areStatePropsEqual }
) {
  let hasRunAtLeastOnce = false
  let state
  let ownProps
  let stateProps
  let dispatchProps
  let mergedProps
  // 如果是第一次运行执行该方法
  function handleFirstCall(firstState, firstOwnProps) {
    state = firstState
    ownProps = firstOwnProps
    stateProps = mapStateToProps(state, ownProps)
    dispatchProps = mapDispatchToProps(dispatch, ownProps)
    mergedProps = mergeProps(stateProps, dispatchProps, ownProps)
    hasRunAtLeastOnce = true
    return mergedProps
  }

  function handleNewPropsAndNewState() {
    stateProps = mapStateToProps(state, ownProps)

    if (mapDispatchToProps.dependsOnOwnProps)
      dispatchProps = mapDispatchToProps(dispatch, ownProps)

    mergedProps = mergeProps(stateProps, dispatchProps, ownProps)
    return mergedProps
  }

  function handleNewProps() {
    if (mapStateToProps.dependsOnOwnProps)
      stateProps = mapStateToProps(state, ownProps)

    if (mapDispatchToProps.dependsOnOwnProps)
      dispatchProps = mapDispatchToProps(dispatch, ownProps)

    mergedProps = mergeProps(stateProps, dispatchProps, ownProps)
    return mergedProps
  }

  function handleNewState() {
    const nextStateProps = mapStateToProps(state, ownProps)
    const statePropsChanged = !areStatePropsEqual(nextStateProps, stateProps)
    stateProps = nextStateProps

    if (statePropsChanged)
      mergedProps = mergeProps(stateProps, dispatchProps, ownProps)

    return mergedProps
  }
  // 处理后续调用
  function handleSubsequentCalls(nextState, nextOwnProps) {
    const propsChanged = !areOwnPropsEqual(nextOwnProps, ownProps)
    const stateChanged = !areStatesEqual(nextState, state)
    state = nextState
    ownProps = nextOwnProps
    // 根据props和state的变动情况执行不同的方法，本质上最后都是调用mergedProps合并props
    if (propsChanged && stateChanged) return handleNewPropsAndNewState()
    if (propsChanged) return handleNewProps()
    if (stateChanged) return handleNewState()
    return mergedProps
  }

  return function pureFinalPropsSelector(nextState, nextOwnProps) {
    return hasRunAtLeastOnce
      ? handleSubsequentCalls(nextState, nextOwnProps)
      : handleFirstCall(nextState, nextOwnProps)
  }
}
```
整体流程比较简单，如果是第一次运行，调用`handleFirstCall`方法，它会根据传入的`state`和`ownProps`来返回`merge`后的`props`，这个方法里面调用了`mapStateToProps(state, ownProps)`，等同于调用了`mapToPropsProxy(state, ownProps)`，然后在判断是`mapStateToProps`还是`mapDispatchToProps`来返回处理后的`props`，后续运行也是一样的，大致的流程都没有什么大的区别，最后可以看一下默认的`mergeProps`是怎么处理的：

```javascript
export function defaultMergeProps(stateProps, dispatchProps, ownProps) {
  // 简单的使用展开运算符合并对象
  return { ...ownProps, ...stateProps, ...dispatchProps }
}
```
可以看到就是直接展开，简明易懂。
