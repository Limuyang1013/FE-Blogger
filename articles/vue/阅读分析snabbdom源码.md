尤大在官宣Vue 2.0的时候这么说过：
>渲染层基于一个轻量级的 Virtual DOM 实现进行了重写，该 Virtual DOM 实现 fork 自 snabbdom。新的渲染层相比 v1 带来了巨大的性能提升，也让 Vue 2.0 成为了最快速的框架之一。
那么对于想要深入了解Vue源码的人来说先深入了解一下snabbdom的实现是有必要的

#### 什么是Virtual DOM

- Virtual DOM使用JavaScript对象来描述节点，只保留一些有用的信息，可以更轻量的描述DOM树的结构
- 将原本需要在真实DOM进行的创建节点,删除节点,添加节点等一系列复杂的DOM操作全部放到Vritual DOM中进行
比如在我们的`snabbdom`中就是用这样的方式来定义一个`VNODE`：

```JavaScript
export interface VNode {
  sel: string | undefined;
  data: VNodeData | undefined;
  children: Array<VNode | string> | undefined;
  elm: Node | undefined;
  text: string | undefined;
  key: Key | undefined;
}

export interface VNodeData {
  props?: Props;
  attrs?: Attrs;
  class?: Classes;
  style?: VNodeStyle;
  dataset?: Dataset;
  on?: On;
  hero?: Hero;
  attachData?: AttachData;
  hook?: Hooks;
  key?: Key;
  ns?: string; // for SVGs
  fn?: () => VNode; // for thunks
  args?: Array<any>; // for thunks
  [key: string]: any; // for any other 3rd party module
}
```
既然DOM树信息可以通过JavaScript对象来表示，反过来我们就可以用这个JavaScript对象还原构建一棵真正的DOM树，当状态变化的时候重新渲染这个 JavaScript 的对象结构，然后将两个JavScript对象进行对比，记录差异，然后把它们应用在真实的DOM树上，这便是`diff`算法

#### 分析源码

从官方给的Demo入手
```JavaScript
var snabbdom = require('snabbdom');
var patch = snabbdom.init([ // Init patch function with chosen modules
  require('snabbdom/modules/class').default, // makes it easy to toggle classes
  require('snabbdom/modules/props').default, // for setting properties on DOM elements
  require('snabbdom/modules/style').default, // handles styling on elements with support for animations
  require('snabbdom/modules/eventlisteners').default, // attaches event listeners
]);
var h = require('snabbdom/h').default; // helper function for creating vnodes

var container = document.getElementById('container');

var vnode = h('div#container.two.classes', {on: {click: someFn}}, [
  h('span', {style: {fontWeight: 'bold'}}, 'This is bold'),
  ' and this is just normal text',
  h('a', {props: {href: '/foo'}}, 'I\'ll take you places!')
]);
// Patch into empty DOM element – this modifies the DOM as a side effect
patch(container, vnode);

var newVnode = h('div#container.two.classes', {on: {click: anotherEventHandler}}, [
  h('span', {style: {fontWeight: 'normal', fontStyle: 'italic'}}, 'This is now italic type'),
  ' and this is still just normal text',
  h('a', {props: {href: '/bar'}}, 'I\'ll take you places!')
]);
// Second `patch` invocation
patch(vnode, newVnode); // Snabbdom efficiently updates the old view to the new state
```
可以看到`snabbdom`核心模块对外暴露一个`init`方法，这个方法接收一个数组，数组里面的每一项都是一个`module`，这些`module`的作用是扩展`snabbdom`生成复杂DOM的能力，可以根据自己的需求决定是否要引入对应的`module`，与此同时你也可以实现自己的`module`，打个比方，如果你的`vnode`节点不需要写入`listener`，你可以不引入`snabbdom/modules/eventlisteners`，调用`init`方法之后会返回一个`patch`函数，这个函数接收两个参数，第一个参数是一个`DOM`节点或者一个用于表示当前视图的`vnode`节点，第二个参数是表示即将要更新的视图的`vnode`节点，`patch`函数会根据传递的参数不同进行对应的`DOM`更新操作，接下来我们可以先看一下`init`函数的核心实现(为了理解方便暂时隐去函数内部的实现细节)：
```JavaScript
export interface Module {
  pre: PreHook;
  create: CreateHook;
  update: UpdateHook;
  destroy: DestroyHook;
  remove: RemoveHook;
  post: PostHook;
}

export function init(modules: Array<Partial<Module>>, domApi?: DOMAPI) {
  // cbs 用于收集 module 中的 hook
  let i: number,
    j: number,
    cbs = {} as ModuleHooks;

  const api: DOMAPI = domApi !== undefined ? domApi : htmlDomApi;

  // 收集 module 中的 hook
  for (i = 0; i < hooks.length; ++i) {
    cbs[hooks[i]] = [];
    for (j = 0; j < modules.length; ++j) {
      const hook = modules[j][hooks[i]];
      if (hook !== undefined) {
        (cbs[hooks[i]] as Array<any>).push(hook);
      }
    }
  }

  function emptyNodeAt(elm: Element) {
    // ...
  }

  function createRmCb(childElm: Node, listeners: number) {
    // ...
  }

  // 创建真正的 dom 节点
  function createElm(vnode: VNode, insertedVnodeQueue: VNodeQueue): Node {
    // ...
  }

  function addVnodes(
    parentElm: Node,
    before: Node | null,
    vnodes: Array<VNode>,
    startIdx: number,
    endIdx: number,
    insertedVnodeQueue: VNodeQueue
  ) {
    // ...
  }

  // 调用 destory hook
  // 如果存在 children 递归调用
  function invokeDestroyHook(vnode: VNode) {
    // ...
  }

  function removeVnodes(parentElm: Node, vnodes: Array<VNode>, startIdx: number, endIdx: number): void {
    // ...
  }

  function updateChildren(parentElm: Node, oldCh: Array<VNode>, newCh: Array<VNode>, insertedVnodeQueue: VNodeQueue) {
    // ...
  }

  function patchVnode(oldVnode: VNode, vnode: VNode, insertedVnodeQueue: VNodeQueue) {
    // ...
  }

  return function patch(oldVnode: VNode | Element, vnode: VNode): VNode {
    // ...
  };
}
```
可以看到首先`init`方法会接收两个参数，一个由`module`组成的数组，以及一个可选参数`domApi`，默认是使用预定义的一些和`DOM`操作相关的`API`，接下来会通过循环遍历收集`modules`中的`hook`，存储到`cbs`对象中，以便在恰当的时机对`DOM`节点进行一些操作，举个例子，我们看一下`class`这个`module`的源码：
```JavaScript
"use strict";
Object.defineProperty(exports, "__esModule", { value: true });
function updateClass(oldVnode, vnode) {
    var cur, name, elm = vnode.elm, oldClass = oldVnode.data.class, klass = vnode.data.class;
    if (!oldClass && !klass)
        return;
    if (oldClass === klass)
        return;
    oldClass = oldClass || {};
    klass = klass || {};
    for (name in oldClass) {
        if (!klass[name]) {
            elm.classList.remove(name);
        }
    }
    for (name in klass) {
        cur = klass[name];
        if (cur !== oldClass[name]) {
            elm.classList[cur ? 'add' : 'remove'](name);
        }
    }
}
exports.classModule = { create: updateClass, update: updateClass };
exports.default = exports.classModule;
```
很明显可以看出这个`module`的作用是对比新旧两个`vnode`节点，更新`classname`，执行的时机分别是在调用`create`和`update`两种`hook`的时候，其他`module`也差不多，这里不展开分析。
下面几个都是功能型函数略过不看，直接看返回的`patch`函数

#### patch函数
```JavaScript
  // 用于dom相关的更新
  return function patch(oldVnode: VNode | Element, vnode: VNode): VNode {
    let i: number, elm: Node, parent: Node;
    const insertedVnodeQueue: VNodeQueue = [];
    // 调用 module 中的 pre hook
    for (i = 0; i < cbs.pre.length; ++i) cbs.pre[i]();

    if (!isVnode(oldVnode)) {
      oldVnode = emptyNodeAt(oldVnode);
    }
    // 判断两个Vnode节点是否相同
    if (sameVnode(oldVnode, vnode)) {
      patchVnode(oldVnode, vnode, insertedVnodeQueue);
    } else {
      elm = oldVnode.elm as Node;
      parent = api.parentNode(elm);

      createElm(vnode, insertedVnodeQueue);

      if (parent !== null) {
        api.insertBefore(parent, vnode.elm as Node, api.nextSibling(elm));
        removeVnodes(parent, [oldVnode], 0, 0);
      }
    }

    for (i = 0; i < insertedVnodeQueue.length; ++i) {
      (((insertedVnodeQueue[i].data as VNodeData).hook as Hooks).insert as any)(insertedVnodeQueue[i]);
    }
    for (i = 0; i < cbs.post.length; ++i) cbs.post[i]();
    return vnode;
  };
```
先通过`isVnode`方法来判断传入的第一个参数是不是`vnode`，如果不是的话通过`emptyNodeAt`方法来将其转换成`vnode`，这两个方法实现都比较简单：
```JavaScript

// isVnode
function isVnode(vnode: any): vnode is VNode {
  // 判断是否具有sel属性
  return vnode.sel !== undefined;
}

// emptyNodeAt
function emptyNodeAt(elm: Element) {
    const id = elm.id ? '#' + elm.id : '';
    const c = elm.className ? '.' + elm.className.split(' ').join('.') : '';
    return vnode(api.tagName(elm).toLowerCase() + id + c, {}, [], undefined, elm);
  }
```
看一眼vnode方法的实现：
```JavaScript
export function vnode(sel: string | undefined,
                      data: any | undefined,
                      children: Array<VNode | string> | undefined,
                      text: string | undefined,
                      elm: Element | Text | undefined): VNode {
  let key = data === undefined ? undefined : data.key;
  return {sel: sel, data: data, children: children,
          text: text, elm: elm, key: key};
}
```
接着往下看，当两个需要被对比的节点都转化成了`vnode`之后，通过`sameVnode`方法判断两个`Vnode`节点是否相同：
```JavaScript
function sameVnode(vnode1: VNode, vnode2: VNode): boolean {
  return vnode1.key === vnode2.key && vnode1.sel === vnode2.sel;
}
```
这里通过判断`key`和`sel`是否相同来判断是否是相同`vnode`节点，如果是两个相同的`vnode`节点，调用`patchVnode`方法来对比更新，如果不是，则通过`createElm`方法创建新的`DOM`节点，如果存在父节点，则通过`htmdomapi`里面的`insertBefore`方法插入新的`DOM`节点到子节点的末尾，然后通过`removeVnodes`移除旧节点完成更新，这里的关键方法是`patchVnode`和`createElm`，简单起见先看第二个。

#### createElm

```JavaScript
  // 通过vnode创建真正的dom节点
  function createElm(vnode: VNode, insertedVnodeQueue: VNodeQueue): Node {
    let i: any, data = vnode.data;
    if (data !== undefined) {
      if (isDef(i = data.hook) && isDef(i = i.init)) {
       // 调用init hook
        i(vnode);
        data = vnode.data;
      }
    }
    let children = vnode.children, sel = vnode.sel;
    // 如果是注释节点
    if (sel === '!') {
      if (isUndef(vnode.text)) {
        vnode.text = '';
      }
      vnode.elm = api.createComment(vnode.text as string);
    } else if (sel !== undefined) {
      // 解析sel
      const hashIdx = sel.indexOf('#');
      const dotIdx = sel.indexOf('.', hashIdx);
      const hash = hashIdx > 0 ? hashIdx : sel.length;
      const dot = dotIdx > 0 ? dotIdx : sel.length;
      const tag = hashIdx !== -1 || dotIdx !== -1 ? sel.slice(0, Math.min(hash, dot)) : sel;
     // 如果有ns属性说明是svg元素
      const elm = vnode.elm = isDef(data) && isDef(i = (data as VNodeData).ns) ? api.createElementNS(i, tag)
                                                                               : api.createElement(tag);
      // 设置元素id
      if (hash < dot) elm.setAttribute('id', sel.slice(hash + 1, dot));
      // 设置元素classname
      if (dotIdx > 0) elm.setAttribute('class', sel.slice(dot + 1).replace(/\./g, ' '));
      // 调用create hook
      for (i = 0; i < cbs.create.length; ++i) cbs.create[i](emptyNode, vnode);
      // 创建子元素节点
      if (is.array(children)) {
        for (i = 0; i < children.length; ++i) {
          const ch = children[i];
          if (ch != null) {
            api.appendChild(elm, createElm(ch as VNode, insertedVnodeQueue));
          }
        }
      } else if (is.primitive(vnode.text)) {
        // 文本节点 
        api.appendChild(elm, api.createTextNode(vnode.text));
      }
      i = (vnode.data as VNodeData).hook; // Reuse variable
      if (isDef(i)) {
        // 调用节点的create hook
        if (i.create) i.create(emptyNode, vnode);
       // insert hook要等dom真正挂载的时候才会调用，这里先存储到数组里面
        if (i.insert) insertedVnodeQueue.push(vnode);
      }
    } else {
      // 文本节点
      vnode.elm = api.createTextNode(vnode.text as string);
    }
    return vnode.elm;
  }
```
创建`DOM`的时候首先调用`vnode`节点的`init hook`，然后判断该`vnode`节点是注释节点/文本节点/带选择器节点的其中之一，至于为什么`sel === '!'`就说明是注释节点这里可以解释一下，`snabbdom`提供了`tovnode`方法来将一个`DOM`节点转换成`vnode`节点，大致用法如下：
```JavaScript
var snabbdom = require('snabbdom')
var patch = snabbdom.init([ // Init patch function with chosen modules
  require('snabbdom/modules/class').default, // makes it easy to toggle classes
  require('snabbdom/modules/props').default, // for setting properties on DOM elements
  require('snabbdom/modules/style').default, // handles styling on elements with support for animations
  require('snabbdom/modules/eventlisteners').default, // attaches event listeners
]);
var h = require('snabbdom/h').default; // helper function for creating vnodes
var toVNode = require('snabbdom/tovnode').default;

var newVNode = h('div', {style: {color: '#000'}}, [
  h('h1', 'Headline'),
  h('p', 'A paragraph'),
]);

patch(toVNode(document.querySelector('.container')), newVNode)
```
这个`tovnode`的源码里面有这么一段：
```JavaScript
 ...
 } else if (api.isComment(node)) {
    text = api.getTextContent(node) as string;
    return vnode('!', {}, [], text, node as any);
  } else {
    return vnode('', {}, [], undefined, node as any);
 ...

```
那么我们再找一下`isComment`方法做了什么：
```JavaScript
function isComment(node: Node): node is Comment {
  return node.nodeType === 8;
}
```
节点类型常量里面type 8就是`Node.COMMENT_NODE`也就是注释节点

然后顺着源码我们这里主要看带选择器节点的创建，首先得到`tagname`、`class`、`id`，然后在根据是否有`ns`属性来决定通过`createElement`还是`createElementNS`来创建节点，紧接着如果存在`children`，循环遍历调用`createElm`方法创建子节点并挂载到父节点上面，`DOM`节点创建完毕之后调用相应的`hookl`即可。

#### patchVnode
`patchVnode`方法会对传入的两个`vnode`节点进行对比，最终对比的结果会体现在`DOM`上，这便是`diff`算法：
```JavaScript
  function patchVnode(oldVnode: VNode, vnode: VNode, insertedVnodeQueue: VNodeQueue) {
    let i: any, hook: any;
    // 调用prepatch hook
    if (isDef(i = vnode.data) && isDef(hook = i.hook) && isDef(i = hook.prepatch)) {
      i(oldVnode, vnode);
    }
    const elm = vnode.elm = (oldVnode.elm as Node);
    let oldCh = oldVnode.children;
    let ch = vnode.children;
    // 如果两个vnode完全相同，直接返回
    if (oldVnode === vnode) return;
    if (vnode.data !== undefined) {
      // 调用 module 上的 update hook
      for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode);
      i = vnode.data.hook;
      // 调用vnode的update方法
      if (isDef(i) && isDef(i = i.update)) i(oldVnode, vnode);
    }
    if (isUndef(vnode.text)) {
      // 新节点存在children
      if (isDef(oldCh) && isDef(ch)) {
        // 新旧节点均存在 children，且不一样时，对 children 进行 diff
        if (oldCh !== ch) updateChildren(elm, oldCh as Array<VNode>, ch as Array<VNode>, insertedVnodeQueue);
      } else if (isDef(ch)) {
        // 旧节点不存在 children 新节点有 children
        // 旧节点存在 text 置空
        if (isDef(oldVnode.text)) api.setTextContent(elm, '');
        addVnodes(elm, null, ch as Array<VNode>, 0, (ch as Array<VNode>).length - 1, insertedVnodeQueue);
      } else if (isDef(oldCh)) {
        // 新节点不存在 children 旧节点存在 children 移除旧节点的 children
        removeVnodes(elm, oldCh as Array<VNode>, 0, (oldCh as Array<VNode>).length - 1);
      } else if (isDef(oldVnode.text)) {
        api.setTextContent(elm, '');
      }
    } else if (oldVnode.text !== vnode.text) {
      if (isDef(oldCh)) {
        // 如果旧的vnode有children则移除
        removeVnodes(elm, oldCh as Array<VNode>, 0, (oldCh as Array<VNode>).length - 1);
      }
      // 更新text
      api.setTextContent(elm, vnode.text as string);
    }
    // 调用 postpatch hook
    if (isDef(hook) && isDef(i = hook.postpatch)) {
      i(oldVnode, vnode);
    }
  }
```
首先调用`vnode`上的`prepatch hook`，然后判断两个`vnode`节点是否完全相同，如果完全相同则直接返回，紧接着遍历调用`module`的`update hook`和`vnode`的`update hook`，然后分别处理如下几种情况：
- 如果新节点不存在`children`而旧节点存在，那么使用`removeVnodes`方法移除旧节点的`children`并且设置文本内容
- 如果新节点存在`children`且旧节点没有，然后移除旧节点的文本内容(如果有的话)，最后通过`addVnodes`方法挂载子节点
- 如果新旧节点均存在`children`，调用`updateChildren`方法对`children`进行`diff`
- 如果新旧节点文本内容不相同，移除旧节点的`children`(如果有的话)，然后更新文本内容
最后当上述步骤都完成之后，调用`vnode`的`postpatch hook`钩子，所以接下来我们需要关注`updateChildren`、`addVnodes`和`removeVnodes`都分别干了什么。

#### addVnodes
```JavaScript
  function addVnodes(parentElm: Node,
                     before: Node | null,
                     vnodes: Array<VNode>,
                     startIdx: number,
                     endIdx: number,
                     insertedVnodeQueue: VNodeQueue) {
    for (; startIdx <= endIdx; ++startIdx) {
      const ch = vnodes[startIdx];
      if (ch != null) {
        // 循环遍历插入dom节点
        api.insertBefore(parentElm, createElm(ch, insertedVnodeQueue), before);
      }
    }
  }
```
主要就是通过遍历`children`数组并且调用`createElm`方法生成`DOM`插入到父元素子节点末尾来达到添加节点的目的

#### removeVnodes
```JavaScript
  function removeVnodes(parentElm: Node,
                        vnodes: Array<VNode>,
                        startIdx: number,
                        endIdx: number): void {
    for (; startIdx <= endIdx; ++startIdx) {
      let i: any, listeners: number, rm: () => void, ch = vnodes[startIdx];
      if (ch != null) {
        // 非文本节点
        if (isDef(ch.sel)) {
          // 调用destroy hook
          invokeDestroyHook(ch);
          // 当所有的remove hook都调用了才会真正调用移除dom的方法
          listeners = cbs.remove.length + 1;
          rm = createRmCb(ch.elm as Node, listeners);
          // 调用module的remove hook
          for (i = 0; i < cbs.remove.length; ++i) cbs.remove[i](ch, rm);
          if (isDef(i = ch.data) && isDef(i = i.hook) && isDef(i = i.remove)) {
            // 调用vnode的remove hook
            i(ch, rm);
          } else {
            rm();
          }
        } else { // Text node
          api.removeChild(parentElm, ch.elm as Node);
        }
      }
    }
  }

  function createRmCb(childElm: Node, listeners: number) {
    return function rmCb() {
      if (--listeners === 0) {
        const parent = api.parentNode(childElm);
        api.removeChild(parent, childElm);
      }
    };
  }
```
同样是遍历，只不过这里是移除节点

#### updateChildren
我们知道当新旧节点均有`children`并且互不相同的时候会调用`updateChildren`方法来对`children`进行`diff`：
```JavaScript
  function updateChildren(parentElm: Node,
                          oldCh: Array<VNode>,
                          newCh: Array<VNode>,
                          insertedVnodeQueue: VNodeQueue) {
    let oldStartIdx = 0, newStartIdx = 0;
    let oldEndIdx = oldCh.length - 1;
    let oldStartVnode = oldCh[0];
    let oldEndVnode = oldCh[oldEndIdx];
    let newEndIdx = newCh.length - 1;
    let newStartVnode = newCh[0];
    let newEndVnode = newCh[newEndIdx];
    let oldKeyToIdx: any;
    let idxInOld: number;
    let elmToMove: VNode;
    let before: any;

    // 遍历 oldCh newCh，对节点进行比较和更新
    // 每轮比较最多处理一个节点，算法复杂度 O(n)
    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
      if (oldStartVnode == null) {
        // 当前节点可能已经被处理过
        oldStartVnode = oldCh[++oldStartIdx]; // Vnode might have been moved left
      } else if (oldEndVnode == null) {
        oldEndVnode = oldCh[--oldEndIdx];
      } else if (newStartVnode == null) {
        newStartVnode = newCh[++newStartIdx];
      } else if (newEndVnode == null) {
        newEndVnode = newCh[--newEndIdx];
      } else if (sameVnode(oldStartVnode, newStartVnode)) {
        // 新旧开始节点相同，直接调用 patchVnode 进行更新，下标向中间推进
        patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue);
        oldStartVnode = oldCh[++oldStartIdx];
        newStartVnode = newCh[++newStartIdx];
      } else if (sameVnode(oldEndVnode, newEndVnode)) {
        // 新旧结束节点相同，直接调用 patchVnode 进行更新，下标向中间推进
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue);
        oldEndVnode = oldCh[--oldEndIdx];
        newEndVnode = newCh[--newEndIdx];
      } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
        // 旧的开始节点等于新的结束节点，两个节点进行patch
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue);
        // 把旧的开始节点插入到末尾
        api.insertBefore(parentElm, oldStartVnode.elm as Node, api.nextSibling(oldEndVnode.elm as Node));
        oldStartVnode = oldCh[++oldStartIdx];
        newEndVnode = newCh[--newEndIdx];
      } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
        // 旧的结束节点等于新的开始节点，两个节点进行patch
        patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue);
        // 把旧的就结束节点插入到开头
        api.insertBefore(parentElm, oldEndVnode.elm as Node, oldStartVnode.elm as Node);
        oldEndVnode = oldCh[--oldEndIdx];
        newStartVnode = newCh[++newStartIdx];
      } else {
        // 如果上述情况都不符合，那么只有可能是以下两种情况
        // 1. 这个节点是新创建的
        // 2. 这个节点在原来的位置是处于中间的（oldStartIdx 和 endStartIdx之间）

        // 如果 oldKeyToIdx 不存在，创建 key 到 index 的映射
        // 而且也存在各种细微的优化，只会创建一次，并且已经完成的部分不需要映射
        if (oldKeyToIdx === undefined) {
          oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx);
        }
        idxInOld = oldKeyToIdx[newStartVnode.key as string];
        if (isUndef(idxInOld)) { // New element
          // 将新节点插入到开头
          api.insertBefore(parentElm, createElm(newStartVnode, insertedVnodeQueue), oldStartVnode.elm as Node);
          newStartVnode = newCh[++newStartIdx];
        } else {
          // 如果不是新节点，说明该节点之前存在，找到该节点
          elmToMove = oldCh[idxInOld];
          // 如果两个节点key相同但是sel不同，那就通过createElm创建新的dom节点然后插入到开头
          if (elmToMove.sel !== newStartVnode.sel) {
            api.insertBefore(parentElm, createElm(newStartVnode, insertedVnodeQueue), oldStartVnode.elm as Node);
          } else {
            // 将oldCh中已存在的节点和新的开始节点进行patch
            patchVnode(elmToMove, newStartVnode, insertedVnodeQueue);
            // 将oldCh中处理过的节点置空，等循环处理到这个地方的时候方便直接跳过
            oldCh[idxInOld] = undefined as any;
            // 将处理过后的节点插入到旧的开始节点之前
            api.insertBefore(parentElm, (elmToMove.elm as Node), oldStartVnode.elm as Node);
          }
          newStartVnode = newCh[++newStartIdx];
        }
      }
    }
    // oldCh或newCh中有一方先循环结束
    if (oldStartIdx <= oldEndIdx || newStartIdx <= newEndIdx) {
      if (oldStartIdx > oldEndIdx) {
        // oldCh 已经全部处理完成，而 newCh 还有新的节点，需要对剩下的每个项都创建新的 dom
        before = newCh[newEndIdx+1] == null ? null : newCh[newEndIdx+1].elm;
        addVnodes(parentElm, before, newCh, newStartIdx, newEndIdx, insertedVnodeQueue);
      } else {
        // newCh 已经全部处理完成，而 oldCh 还有旧的节点，需要将多余的节点移除
        removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx);
      }
    }
  }
```
直接看整个过程会有点懵，可以结合例子理解一下，我们拿两个数组进行演示：
- 假设旧节点顺序为[A, B, C, D]，新节点为[B, A, C, D, E]
![image](https://user-images.githubusercontent.com/11991572/52620314-37302e80-2edf-11e9-956d-57b77b95a2d0.png)
- 第一轮比较：开始结束节点两两并不相等，于是看`newStartVnode`在旧节点中是否存在，最后找到了在第二个位置，调用`patchVnode`进行更新，将 oldCh[1] 至空，将 dom 插入到`oldStartVnode`前面，`newStartIdx`向中间移动，状态更新如下
![image](https://user-images.githubusercontent.com/11991572/52620414-7bbbca00-2edf-11e9-9afd-2b4618e5425b.png)
- 第二轮比较：`oldStartVnode`和`newStartVnode`相等，直接 `patchVnode`，`newStartIdx`和 `oldStartIdx`向中间移动，状态更新如下
![image](https://user-images.githubusercontent.com/11991572/52620536-c9383700-2edf-11e9-8370-2b6293130df1.png)
- 第三轮比较：`oldStartVnode`为空，`oldStartIdx`向中间移动，进入下轮比较，状态更新如下
![image](https://user-images.githubusercontent.com/11991572/52620572-e3721500-2edf-11e9-8fa8-137e36b0a163.png)
- 第四轮比较：`oldStartVnode`和 `newStartVnode`相等，直接`patchVnode`，`newStartIdx`和 `oldStartIdx`向中间移动，状态更新如下
![image](https://user-images.githubusercontent.com/11991572/52620944-fdf8be00-2ee0-11e9-86e4-689e960089c8.png)
- 第五轮比较：`oldStartVnode`和 `newStartVnode`相等，直接`patchVnode`，`newStartIdx` 和 `oldStartIdx`向中间移动，状态更新如下
![image](https://user-images.githubusercontent.com/11991572/52621054-4d3eee80-2ee1-11e9-8935-377ade6c55da.png)
- `oldStartIdx` 已经大于 `oldEndIdx`，循环结束，由于是旧节点先结束循环而且还有没处理的新节点，调用 `addVnodes `处理剩下的新节点

到目前为止，整个snabbdom的核心流程已经梳理完毕

[参考链接](https://juejin.im/post/5b9200865188255c672e8cfd#heading-8)
