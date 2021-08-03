# 组件的本质
组件就是一个函数, 输入一些数据, 渲染对应的内容

一个原始的组件形态:
```js
import { template } from 'lodash'

const MyComponent = props => {
  const compiler = MyComponent.cache ||  (MyComponent.cache = template('<div><%= title %></div>'))
  return compiler(props)
}

// 渲染组件 html 到真实 dom 中
document.getElementById('app').innerHTML = myComponent({ title: 'MyComponent' })
```

这里一个组件的产出, 是一篇它自己的完整的 html 字符串, 而当今现代化的组件, 产出变成了 Virtual Dom (VNode)
一个简单的函数式组件(使用了 snabbdom): 
```js
import { h, init } from 'snabbdom'
const patch = init([])
const MyComponent = props => {
  return h('div', props.title)
}
// 渲染函数式组件
patch(document.getElementById('app'), MyComponent({ title: 'MyComponent' }))
```
一个简单的有状态组件(未使用 snabbdom):
```js
class MyComponent {
  render () {
    return { // 用来描述组件的对象
      tag: 'div'
    }
  }
}
// 这里有个基础的 render 函数, 更为完整的 render 函数在后面
function render (vnode, container) {
  if (typeof vnode.tag !== 'string') {
    vnode = (new vnode.tag()).render()
  }
  container.appendChild(document.createElement(vnode.tag))
}
// 渲染有状态组件
render({ tag: MyComponent }, document.getElementById('app'))
```
Vue 一位作者说:
> 为何组件要从直接产出 html 变成产出 Virtual DOM 呢？其原因是 Virtual DOM 带来了 分层设计，它对渲染过程的抽象，使得框架可以渲染到 web(浏览器) 以外的平台，以及能够实现 SSR 等

> 至于 Virtual DOM 相比原生 DOM 操作的性能，这并非 Virtual DOM 的目标，确切地说，如果要比较二者的性能是要“控制变量”的，例如：页面的大小、数据变化量等

我觉得这个观点还是不太对, 最重要原因应该还是:
  1. 相比操作原生dom 更快 
  2. 相比模板引擎可以记录上一次状态实现diff更新
因为没有这种动力的话, 单凭跨平台的可能性是不足以推动它这么快这么大规模的流行的

函数式组件 与 有状态组件 的区别如下：
 - 函数式组件：
   是一个纯函数
   没有自身状态，只接收外部数据
   产出 VNode 的方式：单纯的函数调用

 - 有状态组件：
   是一个类，可实例化
   可以有自身状态
   产出 VNode 的方式：需要实例化，然后调用其 render 函数

<br>

# 虚拟 DOM
## VNode 的设计
如何描述一个红色背景的正方形 div, 里面有个子节点 span:
```js
const elementVNode = {
  tag: 'div',
  data: {
    style: {
      width: '100px',
      height: '100px',
      backgroundColor: 'red'
    },
  },
  children: [
    { // 一个 span 节点 elementVNode
      tag: 'span',
      data: null,
      children: [
        { // 一个文本节点 textVNode
          tag: null,
          data: null,
          children: '文本内容'
        }
      ]
    }
  ]
}
```
为什么不使用一个 text 属性来描述 vnode 的文本内容呢? 可以, 但是尽可能的在保证语义能够说得通的情况下复用属性, 可以使 vnode 对象更加**轻量**

那么 vnode 如何表示一个非普通标签 (即非原生存在的浏览器元素) 呢? 上节已经实现过, 就是通过检查 tag 属性值是否为 string 类型来区分是否是自定义组件
```js
const componentVNode = {
  tag: MyComponent,
  ...
}

```

除了普通标签和组件之外, 还有两种抽象内容需要描述, Fragment 和 Portal
Fragment: 描述一个 特定的 由若干个 确定 内容组合而成的唯一片段
```html
<!-- 外层这样用 -->
<template>
  <table>
    <tr>
      <Columns />
    </tr>
  </table>
</template>

<!-- Columms Component -->
<template>
  <td></td>
  <td></td>
  <td></td>
</template>
```
可以这样描述 Columns:
```js
// 它的根元素并不是一个实实在在存在的 dom 节点, 而是一个抽象标识
const Fragment = Symbol()
const fragmentVNode = {
  // tag 属性值是一个唯一标识
  tag: Fragment,
  data: null,
  children: [
    {
      tag: 'td',
      data: null
    },
    {
      tag: 'td',
      data: null
    },
    {
      tag: 'td',
      data: null
    }
  ]
}
```
Portal: 允许你把内容渲染到任何地方, 即: 脱离它在 dom 树中的位置关系
比如这样一个组件:
```html
<template>
  <div id="box">
    <!-- Overlay 只能渲染到 #box 下 -->
    <Overlay />
  </div>
</template>
<template>
  <Portal target="#app">
    <!-- 无论在何处使用 Overlay 组件, 它都会渲染到 #app 下面 -->
    <Overlay />
  </Portal>
</template>
```
秘诀是什么呢? 其实和 Fragment 一样, 告诉 render 函数我是 Portal, 那么它就会被渲染到 data 的 target dom 下
```js
const Portal = Symbol()
const portalVNode = {
  tag: Portal,
  data: {
    target: '#app-root'
  },
  children: {
    tag: 'div',
    data: {
      class: 'overlay'
    }
  }
}
```
Flags 优化:
可以注意到在对 vnode 进行各种各样的操作和判断时 (是否是普通标签, 是否有子节点等) 会比较消耗性能
而在生成这些 vnode 时对其进行 flag 信息标注, 每一个 flag 对应一个二进制数, 
后续判断过程中, 通过位运算进行判断可以提高 patch 性能:
```js
const VNodeFlags = {
  // html 标签
  ELEMENT_HTML: 1,
  // SVG 标签
  ELEMENT_SVG: 1 << 1,

  // 普通有状态组件
  COMPONENT_STATEFUL_NORMAL: 1 << 2,
  // 需要被keepAlive的有状态组件
  COMPONENT_STATEFUL_SHOULD_KEEP_ALIVE: 1 << 3,
  // 已经被keepAlive的有状态组件
  COMPONENT_STATEFUL_KEPT_ALIVE: 1 << 4,
  // 函数式组件
  COMPONENT_FUNCTIONAL: 1 << 5,

  // 纯文本
  TEXT: 1 << 6,
  // Fragment
  FRAGMENT: 1 << 7,
  // Portal
  PORTAL: 1 << 8
}

// html 和 svg 都是标签元素，可以用 ELEMENT 表示
VNodeFlags.ELEMENT = VNodeFlags.ELEMENT_HTML | VNodeFlags.ELEMENT_SVG
// 普通有状态组件、需要被keepAlive的有状态组件、已经被keepAlice的有状态组件 都是“有状态组件”，统一用 COMPONENT_STATEFUL 表示
VNodeFlags.COMPONENT_STATEFUL =
  VNodeFlags.COMPONENT_STATEFUL_NORMAL |
  VNodeFlags.COMPONENT_STATEFUL_SHOULD_KEEP_ALIVE |
  VNodeFlags.COMPONENT_STATEFUL_KEPT_ALIVE
// 有状态组件 和  函数式组件都是“组件”，用 COMPONENT 表示
VNodeFlags.COMPONENT = VNodeFlags.COMPONENT_STATEFUL | VNodeFlags.COMPONENT_FUNCTIONAL

// 使用按位与(&)运算
functionalComponentVnode.flags & VNodeFlags.COMPONENT // 真
normalComponentVnode.flags & VNodeFlags.COMPONENT // 真
htmlVnode.flags & VNodeFlags.COMPONENT // 假
```
此外, 为了优化关于 vnode children 子节点的各种情况的判断 (为什么需要判断这些个?):
  - 没有子节点
  - 只有一个子节点
  - 多个子节点
    - 有 key
    - 无 key
  - 不知道子节点的情况
还需要有一个 childFlags:
```js
const ChildrenFlags = {
  // 未知的 children 类型
  UNKNOWN_CHILDREN: 0,
  // 没有 children
  NO_CHILDREN: 1,
  // children 是单个 VNode
  SINGLE_VNODE: 1 << 1,

  // children 是多个拥有 key 的 VNode
  KEYED_VNODES: 1 << 2,
  // children 是多个没有 key 的 VNode
  NONE_KEYED_VNODES: 1 << 3
}

// 派生出一个"多节点"的标识
ChildrenFlags.MULTIPLE_VNODES = ChildrenFlags.KEYED_VNODES | ChildrenFlags.NONE_KEYED_VNODES

// 这样我们判断一个 VNode 的子节点是否是多个子节点就变得容易多了
someVNode.childFlags & ChildrenFlags.MULTIPLE_VNODES
```

不同类型的 vnode 会采用不同的设计, 现在总结一个 vnode 的种类:
  - html/svg 元素
  - 组件 Component
    - 函数式组件
    - 有状态组件
      - 普通有状态组件
      - 需要被 keepAlive 的有状态组件
      - 已经被 keepAlive 的有状态组件
  - 纯文本 text
  - Fragment
  - Portal

一个基本的 vnode 设计:
```ts
export interface VNode {
  // _isVNode 属性在上文中没有提到，它是一个始终为 true 的值，有了它，我们就可以判断一个对象是否是 VNode 对象
  _isVNode: true
  // el 属性在上文中也没有提到，当一个 VNode 被渲染为真实 DOM 之后，el 属性的值会引用该真实DOM
  el: Element | null
  flags: VNodeFlags
  tag: string | FunctionalComponent | ComponentClass | null
  data: VNodeData | null
  children: VNodeChildren
  childFlags: ChildrenFlags
}
```
data:
任何可以对这个 vnode 内容的描述都可以放进 data 中, 比如样式, 事件, 属性等

_isVNode: 
一个始终为 true 的值, 可以判断一个对象是否是 VNode 对象

el:
一个 vnode 应该有一个 el 属性, 它在 vnode 被渲染为真实 dom 之前一直是 null, 被渲染到真实 dom 后会变成该真实 dom 元素的引用

## h 函数
无论是模板还是 jsx 语法都需要经过编译 (运行时编译 or 构建时编译), 那么是直接编译成 vnode 树好呢? 还是编译成由 h 函数组成的运行时调用集合好呢?
很难说
如果编译成 vnode tree, 那么它的更新会变得像 dom 树一样困难
如果将 公用, 灵活, 复杂的逻辑封装成函数, 并交给运行时, 可以进行灵活的 patch

一个简单的 h 函数, 生产一个 vnode:
```js
function h(tag, data = null, children = null) {
  // 在 vnode 被创建时, 确定好其 flags
  let flags = null
  if (typeof tag === 'string') { // 普通标签节点
    flags = tag === 'svg' ? VnodeFlags.VNodeFlags.ELEMENT_SVG : VNodeFlags.ELEMENT_HTML
  } else if (tag === Fragment) { // fragment 节点
    flags = VNodeFlags.FRAGMENT
  } else if (tag === Portal) { // portal 节点
    flags = VNodeFlags.PORTAL
    tag = data && data.target
  } else { // 组件节点
    // 兼容 Vue2 的对象式组件
    // Vue2 中的组件是用一个对象描述
    if (tag !== null && typeof tag === 'object') {
      flags = tag.functional // 判断是否是函数式组件
        ? VNodeFlags.COMPONENT_FUNCTIONAL       // 函数式组件
        : VNodeFlags.COMPONENT_STATEFUL_NORMAL  // 有状态组件
    } else if (typeof tag === 'function') {
      // Vue3 的类组件
      // Vue3 中的有状态组件是一个继承了基类的类, 用原型上是否有 render 盘算是否是函数式组件
      flags = tag.prototype && tag.prototype.render
        ? VNodeFlags.COMPONENT_STATEFUL_NORMAL  // 有状态组件
        : VNodeFlags.COMPONENT_FUNCTIONAL       // 函数式组件
    }
  } 
  // else { // 为什么不考虑文本节点的情况呢?
  // 对于 text 节点的情况, 我们一般不用 h 函数创建, 而是直接写字符串了
  // 然后在对 vnode children 的遍历中检测到该字符串的寓意是文本节点的时候会为其自动创建一个纯文本的 VNode 对象
  // function createTextVNode(text) {
  //   return {
  //     _isVNode: true,
  //     // flags 是 VNodeFlags.TEXT
  //     flags: VNodeFlags.TEXT,
  //     tag: null,
  //     data: null,
  //     // 纯文本类型的 VNode，其 children 属性存储的是与之相符的文本内容
  //     children: text,
  //     // 文本节点没有子节点
  //     childFlags: ChildrenFlags.NO_CHILDREN,
  //     el: null
  //   }
  // }
  // }

  // TODO: 其实还需要根据 传入的 children 参数 确定 childFlags
  
  return {
    _isVNode: true,
    flags, tag, data, children,
    childFlags: null, // ChildrenFlags.NO_CHILDREN,
    el: null
  }
}
```
需要注意的是, 如果创建的一个 组件类型 的 vnode, 它没有 children, 所有的子节点都应该转化为 slots
同时相应的, 它也没有 childFlags了
至于其他节点为什么需要 childFlags? slots 又是什么, 没弄明白, 需要继续学习

# 渲染器
所谓渲染器 (renderer) 就是将 virtual dom 渲染成某个特定平台下真实 dom 的工具
一个基本的 render 函数示例:
```js
function render(vnode, container) {
  const prevVNode = container.vnode
  if (prevVNode == null) {
    if (vnode) {
      // 没有旧的 VNode，只有新的 VNode。使用 `mount` 函数挂载全新的 VNode
      mount(vnode, container)
      // 将新的 VNode 添加到 container.vnode 属性下，这样下一次渲染时旧的 VNode 就存在了
      container.vnode = vnode
    }
  } else {
    if (vnode) {
      // 有旧的 VNode，也有新的 VNode。则调用 `patch` 函数打补丁
      patch(prevVNode, vnode, container)
      // 更新 container.vnode
      container.vnode = vnode
    } else {
      // 有旧的 VNode 但是没有新的 VNode，这说明应该移除 DOM，在浏览器中可以使用 removeChild 函数。
      container.removeChild(prevVNode.el)
      container.vnode = null
    }
  }
}
```
除此之外, 渲染函数还可以负责以下工作:
* 控制组件部分生命周期钩子的调用
* 抹平多平台渲染 dom 操作差异
* 实现异步渲染
* (核心) 实现 diff 算法


## mount
将旧的 各种类型的 vnode 挂载成全新的真实 DOM:
```js
function mount(vnode, container) {
  const { flags } = vnode
  if (flags & VNodeFlags.ELEMENT) 
    mountElement(vnode, container)
  else if (flags & VNodeFlags.COMPONENT)
    mountComponent(vnode, container)
  else if (flags & VNodeFlags.TEXT)
    mountText(vnode, container)
  else if (flags & VNodeFlags.FRAGMENT)
    mountFragment(vnode, container)
  else if (flags & VNodeFlags.PORTAL)
    mountPortal(vnode, container)
}

function mountElement(vnode, container) {
  vnode.el = document.createElement(vnode.tag)

  // 处理 data 中的属性
  Object.keys(vnode.data).forEach(key => {
    switch(key) {
      case 'style':
        for (let k in vnode.data.style) {
          vnode.el.style[k] = vnode.data.style[k]
        }
        break
      case 'class':
        vnode.el.className = vnode.data.class
        break
      case 'on': // 事件如何更新?
        let events = vnode.data.on
        Object.keys(events).forEach(event => {
          vnode.el.addEventlistener(event, events[event])
        })
        break
      default:
        if (/\[A-Z]|^(?:value|checked|selected|muted)$/.test(key)) {
          vnode.el[key] = vnode.data[key]
        } else { // 因为 setAttribute 函数会将任何类型 value 转为字符串设置在 dom 上
          el.setAttribute(key, vnode.data[key])
        }
        break
    }
  })

  // 处理子节点
  if (vnode.childFlags & ChildrenFlags.SINGLE_VNODE) {
    mount(children, vnode.el)
  } else if (vnode.childFlags & ChildrenFlags.MUTIPLE_VNODES) {
    vnode.children.forEach(child => {
      mount(child, vnode.el)
    })
  }ß
  container.appendChild(vnode.el)
}

function mountText(vnode, container) { // text 节点的 字符串会被 h 函数存到它的 children 属性中
  vnode.el = document.createTextNode(vnode.children)
  container.appendChild(vnode.el)
}

function mountFragment(vnode, container) {
  const { children, childFlags } = vnode
  switch (childFlags) {
    case ChildrenFlags.SINGLE_VNODE:
      // 如果是单个子节点，则直接调用 mount
      mount(children, container)
      vnode.el = children.el
      break
    case ChildrenFlags.NO_CHILDREN:
      // 如果没有子节点，等价于挂载空片段，会创建一个占位的空的文本节点占位
      const placeholder = createTextVNode('') // 上文提到过的一个创建文本 vnode 的函数
      mountText(placeholder, container)
      vnode.el = placeholder.el
      break
    default:
      // 多个子节点，遍历挂载之
      for (let i = 0; i < children.length; i++) {
        mount(children[i], container)
      }
      vnode.el = children[0].el
  }
}

function mountPortal(vnode, container) {
  const { tag, children, childFlags } = vnode
  const taget = typeof tag === 'string' ? document.querySelector(tag) : tag
  if (childFlags & ChildrenFlags.SINGLE_VNODE) {
    mount(children, target)
  } else if (childFlags & ChildrenFlags.MUTIPLE_VNODES) {
    children.forEach(child => {
      mount(child, target)
    })
  }

  // 虽然 portal 的节点看起来和 container 无关了
  // 但是它的行为仍然应该像普通的 dom 元素一样, 如事件的捕获/冒泡机制仍然按照代码所编写的 dom 结构实施
  // 要实现这个功能就必须需要一个占位的 dom 元素来承接事件
  // 依然是用空文本节点来占位
  const placeholder = createTextVNode('')
  // 将该节点挂载到 container 中
  mountText(placeholder, container, null)
  // el 属性引用该节点
  vnode.el = placeholder.el
}

function mountComponent(vnode, container) {
  function mountStatefulComponent(vnode, container) {
    const instance = (vnode.children = new vnode.tag()) // 为什么可以放 children 里? 因为组件没有 children
    // instance.$vnode = instance.render()
    // mount(instance.$vnode, container)
    // instance.$el = vnode.el = instance.$vnode.el
    // 疑问: 这里为什么需要套一个 instance.$vnode ?
    // 答案: 为了有状态组件的主动更新 (可以看做 patchStatefulComponent 功能的一部分)
    instance._update = function() {
      if (instance._mounted) { // 已挂载的组件进来就是更新
        const prevVNode = instance.$vnode // 1、拿到旧的 VNode
        const nextVNode = (instance.$vnode = instance.render()) // 2、重渲染新的 VNode
        patch(prevVNode, nextVNode, prevVNode.el.parentNode) // 3、patch 更新
        instance.$el = vnode.el = instance.$vnode.el // 4、更新 vnode.el 和 $el
      } else { // 未挂载的组件进来 第一次 挂载
        instance.$vnode = instance.render() // 1、渲染VNode
        mount(instance.$vnode, container) // 2、挂载了
        instance._mounted = true // 3、组件已挂载的标识
        instance.$el = vnode.el = instance.$vnode.el // 4、el 属性值 和 组件实例的 $el 属性都引用组件的根DOM元素
        instance.mounted && instance.mounted() // 5、调用 mounted 钩子 (钩子中异步调用 _update)
      }
    }

    instance._update()
  }

  function mountFunctionalComponent(vnode, container) {
    // 由于函数式组件不能生成实例, 所以可以给 vnode 进行一个扩展
    vnode.handle = {
      prev: null,
      next: vnode,
      container,
      update () {
        // 这个 vnode 就将一直是第一次挂载时的始祖 vnode
        if (vnode.handle.prev) { // 说明被挂载过了, 现在需要更新了
          const prevVNode = vnode.handle.prev
          const nextVNode = vnode.handle.next
          const $prevVNode = prevVNode.children // prevTree 是组件产出的旧的 VNode

          // 开始更新 (这里真TM绕)
          const props = nextVNode.data
          const $nextVNode = (nextVNode.children = nextVNode.tag(props))
          patch($prevVNode, $nextVNode, vnode.handle.container)
        } else {
          const props = vnode.data // 假装 props
          const $vnode = (vnode.children = vnode.tag(props))
          mount($vnode, container)
          vnode.el = $vnode.el
        }
      }
    }

    vnode.handle.upadate()
  }

  if (vnode.flags & VNodeFlags.COMPONENT_STATEFUL) {
    mountStatefulComponent(vnode, container)
  } else {
    mountFunctionalComponent(vnode, container)
  }
}
```

## patch
patch 就是打补丁, 如果旧的 vnode 存在, 则会进行新旧 vnode 的对比, *试图*以最小的资源开销完成 DOM 的更新
patch 一般发生在 re-render 的时候, 毕竟头一次渲染直接 mount 就行了
re-render 始于数据状态变更而引起的组件变更
```js
const prevVNode = h('div')

class MyComponent {
  render () {
    return h('h1', null, '我是一个新的 vnode')
  }
}

const nextVNode = h(MyComponent) // 由 h 函数封装一个组件类型的 vnode 出来

// 第一次渲染 vnode 到 #app，此时会调用 mount 函数
render(prevVNode, document.getElementById('app'))

// 第二次渲染新的 vnode 到相同的 #app 元素，此时会调用 patch 函数
render(nextVNode, document.getElementById('app'))

function patch(prevVNode, nextVNode, container) {
  const prevFlags = prevVNode.flags
  const nextFlags = nextVNode.flags

  if (prevFlags !== nextFlags) { // 不同种类的节点直接替换了
    replaceVNode(prevVNode, nextVNode, container)
  } else if (nextFlags & VNodeFlags.ELEMENT) {
    patchElement(prevVNode, nextVNode, container)
  } else if (nextFlags & VNodeFlags.TEXT) {
    patchText(prevVNode, nextVNode, container)
  } else if (nextFlags & VNodeFlags.FRAGEMENT) {
    patchFragment(prevVNode, nextVNode, container)
  } else if (nextFlags & VNodeFlags.PORTAL) {
    patchPortal(prevVNode, nextVNode, container)
  } else if (nextFlags & VNodeFlags.COMPONENT) {
    patchComponent(prevVNode, nextVNode, container)
  }
}

function replaceVNode(prevVNode, nextVNode, container) {
  container.removeChild(prevVNode.el)

  // 如果是有状态组件类型, 在挂载之前, 需要做一个特殊处理
  if (preVNode.flags & VNodeFlags.COMPONENT_STATEFULE_NORMAL) {
    // 有状态组件的 children 就是其组件的 instance 实例
    prevVNode.children.unmounted() // 调用它的 unmounted 钩子, 与 mounted 钩子呼应
  }

  mount(nextVNode, container)
}

function patchElement(prevVNode, nextVNode, container) {
  if (prevVNode.tag !== nextVNode.tag) { // 不同 tag 的普通标签直接替换了
    replaceVNode(prevVNode, nextVNode, container)
    return 
  }

  // 如果是同一个 tag, 说明两个 vnode 的差异只在于 data 和 children 了
  const el = nextVNode.el = prevVNode.el
  const prevData = prevVNode.data
  const nextData = nextVNode.data
  const prevChildFlags = prevVNode.childFlags
  const nextChildFlags = nextVNode.childFlags
  const prevChildren = prevVNode.prevChildren
  const nextChildren = nextVNode.nextChildren

  // 处理 data 
  nextData && Object.keys(nextData).forEach(key => {
    const prevValue = prevData[key]
    const nextValue = nextData[key]

    switch(key) {
       case 'style':
        for (let k in nextValue) {
          el.sytle[k] = nextValue[k]
        }
        for (let k in prevValue) {
          if (!nextValue.hasOwnProperty(k)) {
            el.style[k] = ''
          }
        }
        break
      case 'class':
        el.className = nextValue
        break
      case 'on':
        const prevEvents = prevValue
        const nextEvents = nextValue
        // 先把所有之前事件删除
        Object.keys(prevEvents).forEach(event => {
          el.removeEventListener(event, prevEvents[event])
        })
        // 再把新的事件统一添加
        Object.keys(nextEvents).forEach(event => {
          el.addEventlistener(event, nextEvents[event])
        })
        break
      default:
        if (/\[A-Z]|^(?:value|checked|selected|muted)$/.test(key)) {
          el[key] = nextValue
        } else { // 因为 setAttribute 函数会将任何类型 value 转为字符串设置在 dom 上
          el.setAttribute(key, nextValue)
        }
        break
    }
  })

  // 处理 children
  if (prevChildFlag & ChildrenFlags.SINGLE_VNODE) {
    if (nextChildFlag & ChildrenFlags.SINGLE_VNODE) {
      // 两个单独子节点 patch
      patch(prevChildren, nextChildren, el)
    } else if (nextChildFlag & ChildrenFlags.NO_CHILDREN) {
      // 移除旧的已渲染子节点
      // 这里如果 chidlren 是 fragment 类型, 则需要做额外处理把 fragment 渲染的一堆元素移出
      el.removeChild(prevChildren.el)
    } else if (nextChildFlag & ChildrenFlags.MUTIPLE_VNODE) {
      // 移除旧子节点, 挂载新子节点
      el.removeChild(prevChildren.el)
      nextChildren.forEach(child => {
        mount(child, el)
      })
    }
  } else if (prevChildFlag & ChildrenFlags.NO_CHILDREN) {
    if (nextChildFlag & ChildrenFlags.SINGLE_VNODE) {
      // 挂载新的子节点
      mount(nextChildren, el)
    } else if (nextChildFlag & ChildrenFlags.NO_CHILDREN) {
      // 什么都不做
    } else if (nextChildFlag & ChildrenFlags.MUTIPLE_VNODE) {
      // 挂载新的多个子节点
      nextChildren.forEach(child => {
        mount(child, el)
      })
    }
  } else if (prevChildFlag & ChildrenFlags.MUTIPLE_VNODE) {
    if (nextChildFlag & ChildrenFlags.SINGLE_VNODE) {
      // 移除旧的多个子节点, 增加新的单个节点
      prevChidren.forEach(child => {
        el.removeChild(child.el)
      })
      mount(nextChildren, el)
    } else if (nextChildFlag & ChildrenFlags.NO_CHILDREN) {
      // 移除旧的多个子节点
       prevChidren.forEach(child => {
        el.removeChild(child.el)
      })
    } else if (nextChildFlag & ChildrenFlags.MUTIPLE_VNODE) {     
      // 这里暂时先使用一种简单方法:
      for (let i = 0; i < prevChildren.length; i++) {
        container.removeChild(prevChildren[i].el)
      }
      // 遍历新的子节点，将其全部添加
      for (let i = 0; i < nextChildren.length; i++) {
        mount(nextChildren[i], container)
      }
      // 这里为什么不 推荐使用 移除所有旧节点 添加多个新节点 的方式呢?
      // 虽然前面的 patch 都是 0/1 对 n, 这种情况没必要费劲 直接替换了
      // 然而 n 对 n 是可以优化的, 即 children 是可以复用的
      // 核心的 diff (同层级比较) 算法移步下文 #diff
    }
  }
}

function patchText(prevVNode, nextVNode, container) {
  const el = (nextVNode.el = prevVNode.el)
  if (nextVNode.children !== prevVNOde.children) { // 文本节点的 children 是一个字符串
    replaceVNode(prevVNode, nextVNode, container)
    // el.nodeValue = nextVNode.children // 只适用于浏览器环境 DOM API
  }
}

function patchFragment(prevVNode, nextVNode, container) {
  // ... 这里直接与 patchElement 一样的方式 patch children
  // patchChildren(
  //   prevVNode.childFlags, // 旧片段的子节点类型
  //   nextVNode.childFlags, // 新片段的子节点类型
  //   prevVNode.children,   // 旧片段的子节点
  //   nextVNode.children,   // 新片段的子节点
  //   container
  // )
  // 然后与 mountFragment 时一样, 根据内部的 子节点情况不同, el 也不同
  switch(nextVNode.childFlags) {
    case ChildrenFlags.SINGLE_VNODE:
      nextVNode.el = nextVNode.children.el
      break
    case ChilrenFlags.NO_CHILDREN:
      nextVNode.el = prevVNode.el
      break
    default: // mutiple vnodes
      nextVNode.el = nextVNode.children[0].el
  }
}

function patchPortal(prevVNode, nextVNode, container) {
  // ... 这里直接与 patchElement 一样的方式 patch children
  // patchChildren(
  //   prevVNode.childFlags, // 旧片段的子节点类型
  //   nextVNode.childFlags, // 新片段的子节点类型
  //   prevVNode.children,   // 旧片段的子节点
  //   nextVNode.children,   // 新片段的子节点
  //   prevVNode.tag // 注意容器元素是旧的 container
  // )

  // 对于 Portal 类型的 vnode 来说 el 始终是一个占位的文本节点
  nextVNode.el = prevVNode.el

  // 如果新旧容器不同，才需要搬运 (前面 patch 到了旧的 tag 上, 现在从旧的 tag 搬运到新的 tag)
  if (nextVNode.tag !== prevVNode.tag) {
    // 获取新的容器元素，即挂载目标
    const container = typeof nextVNode.tag === 'string'
        ? document.querySelector(nextVNode.tag)
        : nextVNode.tag

    switch (nextVNode.childFlags) {
      case ChildrenFlags.SINGLE_VNODE:
        // 如果新的 Portal 是单个子节点，就把该节点搬运到新容器中
        container.appendChild(nextVNode.children.el)
        break
      case ChildrenFlags.NO_CHILDREN:
        // 新的 Portal 没有子节点，不需要搬运
        break
      default:
        // 如果新的 Portal 是多个子节点，遍历逐个将它们搬运到新容器中
        for (let i = 0; i < nextVNode.children.length; i++) {
          container.appendChild(nextVNode.children[i].el)
        }
        break
    }
  }
}

function patchComponent(prevVNode, nextVNode, container) {
  function patchStatefulComponent(prevVNode, nextVNode, container) {
    // 这种情况只会在组件的 被动更新 时发生
    const instance = (nextVNode.children = prevVNode.children) // 1、获取组件实例
    // 简单化直接写 data 实际应从 data 中分离出给子组件的 props
    instance.$props = nextVNode.data // 2、更新 props
    instance._update() // 3、更新组件
  }

  function patchFunctionalComponent(prevVNode, nextVNode, container) {
    // 与 mountFunctionalComponent 相对应
    const handle = (nextVNode.handle = prevVNode.handle)
    handle.prev = prevVNode
    handle.next = nextVNode
    handle.container = container
    handle.update()
  }

  if (nextVNode.tag !== prevVNode.tag) {
    replaceVNode(prevVNode, nextVNode, container)
  } else if (nextVNode.flags & VNodeFlags.COMPONENT_STATEFUL_NORMAL) {
    patchStatefulComponent(prevVNode, nextVNode, container)
  } else {
    patchFunctionalComponent(prevVNode, nextVNode, container)
  }
}
```

<br>

## diff
其实 patch 过程在很多地方就是指代 diff, 这个 diff 是相对于上面的 "简单 diff" 的 "核心 diff" 算法
这里认为, 只有在有多个旧 vnode 节点和多个新 vnode 节点的情况下, 才需要使用这种同层级 diff 算法
为什么呢?
举一个例子, 比如经常遇到的可排序的列表:
```html
<ul>
  <li>1</li>
  <li>2</li>
  <li>3</li>
</ul>
```
使用一个 vnode 数组来表示这个 ul 的 children
```js
[
  h('li', null, 1),
  h('li', null, 2),
  h('li', null, 3)
]
// 接着由于数据变化导致了列表的数据发生了改变
[
  h('li', null, 3),
  h('li', null, 1),
  h('li', null, 2)
]
```
这个时候如果使用上文代码中的 "简单 diff" 方式, 先删除所有的 children, 再新建所有的 children
这样就没有 复用 任何 DOM
怎么实现 DOM 的复用呢? 直观的想法是 一一对比两个 ul 的 children:
```js
...
// in ul vnode patch
for (let i = 0; i < prevChildren.length; ++i) {
  patch(prevChildren[i], nextChildren[i], container)
}
...
```
这样每一个 children vnode 会和 同一位置上的对应 vnode 进行更新
节省了 移除DOM 和 新建DOM 的开销
但是很多时候, prevChildren 和 nextChildren 并不等长, 所以优化一下:
```js
...
// in ul vnode patch function:
const prevLen = prevChildren.length
const nextLen = nextChildren.length

// 先遍历短的
let len = prevLen < nextLen ? prevLen : nextLen
for (let i = 0; i < len; ++i) {
  patch(prevChildren[i], nextChildren[i], container)
}
if (prevLen < nextLen) { // 少补
  for (let i = len; i < nextChildren.length; ++i) {
    mount(nextChildren[i], container)
  }
} else if (prevLen > nextLen) { // 多退
  for (let i = len; i < prevLen; ++i) {
    container.removeChild(prevChildren[i].el)
  }
}
...
```
到这里 DOM 已经可以复用了, 但是还有什么问题吗? 还有
我们的新旧两个 children vnode 集合是一一对比然后更新的, 但是有时似乎没有必要
有的 vnode 只有顺序是不同的, 如果只是修改其顺序, DOM 操作的开销可以进一步节省
要做的移动 vnode, 就需要新增一个 key 来唯一标识一个 vnode
```js
// key 可以保存在 data 中
h('li', { key: 'a' }, 123)

// h 函数
export function h(tag, data = null, children = null) {
  // 省略...
  return {
    _isVNode: true,
    flags,
    tag,
    data,
    key: data && data.key ? data.key : null, // 这里增加 key
    children,
    childFlags,
    el: null
  }
}
```
就可以写出一个新的算法 (时间复杂度 O(n^2)):
```js
let lastIndex = 0 // 表示寻找过程中遇到的最大索引值
for (let i = 0; i < nextChildren.length; ++i) {
  const nextVNode = nextChildren[i]
  let j = 0, find = false
  // 遍历旧的 children
  for (j; j < prevChildren.length; j++) {
    const prevVNode = prevChildren[j]
    // 如果找到了具有相同 key 值的两个节点，则调用 `patch` 函数更新之
    if (nextVNode.key === prevVNode.key) {
      find = true // 标识找到了, 用以区别出旧 vnodes 中不存在的 新 vnodes
      patch(prevVNode, nextVNode, container) // 如何新旧无变化则 patch 函数等于什么都不会做
      if (j < lastIndex) {
        // 说明这个 prevVNode 所对应的 真实 DOM 需要移动
        const refNode = nextChildren[i - 1].el.nextSibling // 没有 nextSibling 会怎样呢?
        container.insertBefore(prevVNode.el, refNode)
        break // break 为什么放这里?
      } else {
        // 说明 lastIndex 需要更新
        lastIndex = j
      }
    }
  }
  if (!find) { // 没找到就当新节点挂载之
    // 这里其实不能直接使用 mount 方法
    // mount(nextVNode, container)
    // 为什么呢?
    // 因为我们的 mountElement 使用的是 appendChild 会把这个 vnode 插入 container 的末尾
    // 而这个 vnode 实际应该在前一个 vnode 所对应的真实 DOM 之后
    // 我们得先拿到一个可以 insertBefore 的 node 对象
    // 如果当前新节点是第一个, 那么我们就往旧节点第一个之前插
    // 如果不是第一个, 我们往当前节点的 前一个节点的 对应真实 DOM 的 后一个节点 之前插
    const refNdoe = i - 1 < 0 ? prevChildren[0].el : nextChildren[i - 1].el.nextSibling
    container.insertBefore(nextVNode.el, refNode) // 这个也可以集成进 mountElement 里面去
  }
}
// 在新的节点遍历结束后, 再遍历一遍节点, 删除新节点中不存在的节点
prevChildren.forEach(pervVNode => {
   if (!nextChildren.find(vnode => vnode.key === prevVNode.key)) { // if 没找到
    container.removeChild(prevVNode.el)
  }
})
```
思考一下: 什么时候 (或者说哪些 vnode) 需要添加 key 属性呢?
应该是所有具有相同父节点的 子元素 (children vnodes) 都应该添加 key, 以实现正确的 DOM 复用
而不添加 key 属性的话 (key 为 null or undefined), 有两种处理方式, 
一种是当做没有 key 处理, 另一种是当做相同 key 处理, 哪一种更合理呢?

以上这个就是 React 所采用的 diff 算法, 但是这个算法仍然可以继续优化

### 双端比较法
在前面的时候已经感觉到了这种算法的不足: 只能从左到右让 旧节点 一个一个地 向后 挪动
那么能不能让节点也可以从后往前移动呢?
```js
// 双端比较 diff 算法需要定义一堆指针
let oldStartIdx = 0
let oldEndIdx = prevChildren.length - 1
let newStartIdx = 0
let newEndIdx = nextChildren.length - 1
let oldStartVNode = prevChildren[oldStartIdx]
let oldEndVNode = prevChildren[oldEndIdx]
let newStartVNode = nextChildren[newStartIdx]
let newEndVNode = nextChildren[newEndIdx]

// 开始编织
while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
  // 一次比较过程有四个步骤
  // 如果在某一个步骤中找到了可复用节点, 我们就需要将相应的位置指针 后移/前移 一位
  if (!oldStartVNode) { // 可能是 undefined
    oldStartVNode = prevChildren[++oldStartIdx]
  } else if (!oldEndVNode) { // 可能是 undefined
    oldEndVNode = prevChildren[--oldEndIdx]
  } else if (oldStartVNode.key === newStartVNode.key) { // 步骤一：oldStartVNode 和 newStartVNode 比对
    patch(oldStartVNode, newStartVNode, container) // 同一索引位置, 直接更新
    // 更新索引
    oldStartVNode = prevChildren[++oldStartVNode]
    newStartVNode = nextChildren[++newStartIdx]
  } else if (oldEndVNode.key === newEndVNode.key) { // 步骤二：oldEndVNode 和 newEndVNode 比对
    patch(oldEndVNode, newEndVNode, container) // 同一索引位置, 直接更新
    // 更新索引
    oldEndVNode = prevChildren[--oldEndVNode]
    newEndVNode = nextChildren[--newEndIdx]
  } else if (oldStartVNode.key === newEndVNode.key) { // 步骤三：oldStartVNode 和 newEndVNode 比对
    patch(oldStartVNode, newEndVNode, container) // 先更新
    // 将旧节点 真实 DOM 前面的移动到后面
    container.insertBefore(oldStartVNode.el, oldEndVNode.el.nextSibling)
    // 更新索引
    newEndVNode = prevChildren[--newEndIdx]
    oldStartVNode = nextChildren[++oldStartIdx]
  } else if (oldEndVNode.key === newStartVNode.key) { // 步骤四：oldEndVNode 和 newStartVNode 比对
    patch(oldEndVNode, newStartVNode, container) // 先更新
    // 将旧节点 真实 DOM 后面的插到前面
    container.insertBefore(oldEndVNode.el, oldStartVNode.el)
    // 更新索引
    oldEndVNode = prevChildren[--oldEndIdx]
    newStartVNode = nextChildren[++newStartIdx]
  } else { // 四个步骤都没有匹配成功的情况
    // 遍历旧 children，试图寻找与 newStartVNode 拥有相同 key 值的元素
    const idxInOld = prevChildren.findIndex(
      node => node.key === newStartVNode.key // 注意这里比的是 newStartVNode
    )

    if (idxInOld >= 0) { // 找到了
      // vnodeToMove 就是在旧 children 中找到的节点，该节点所对应的真实 DOM 应该被移动到最前面
      const vnodeToMove = prevChildren[idxInOld]
      // 调用 patch 函数完成更新
      patch(vnodeToMove, newStartVNode, container)
      // 把 vnodeToMove.el 移动到最前面，即 oldStartVNode.el 的前面
      container.insertBefore(vnodeToMove.el, oldStartVNode.el)
      // 由于旧 children 中该位置的节点所对应的真实 DOM 已经被移动，所以将其设置为 undefined
      prevChildren[idxInOld] = undefined // 反正 本来 prevChildren 马上也没用了, 即将被回收
    } else { // 没找到, 说明是新增元素节点
      // 这时的当前元素一定是 newStartVNode, 因为前面 idxInOld 比的就是 newStartVNode
      container.insertBefore(newStartVNode.el, oldStartVNode.el)
    }

    // 无论如何都将 newStartIdx 下移一位, 否则死循环
    newStartVNode = nextChildren[++newStartIdx]
  }
}

//  循环结束后, 需要检查有没有漏掉的 该新增的新节点 或 该删除的旧节点
if (oldEndIdx < oldStartIdx) {
  // 添加新节点
  for (let i = newStartIdx; i <= newEndIdx; i++) {
    container.insertBefore(nextChildren[i].el, oldStartVNode.el)
  }
} else if (newEndIdx < newStartIdx) {
  for (let i = oldStartIdx; i <= oldEndIdx; i++) {
    container.removeChild(prevChildren[i].el)
  }
}
```
双端比较在移动 DOM 方面更具有优势, 有点类似于快速排序的思路, 不会因为 DOM 结构的差异而产生影响
这也是 Vue2.x 所采用的算法, 借鉴于开源库 [snabbdom](https://github.com/snabbdom/snabbdom)

Vue3 中, 又采用了一种新的 核心 Diff 算法, 借鉴于 
[ivi](https://github.com/localvoid/ivi) 和
[inferno](https://github.com/infernojs/inferno)

### inferno
inferno 本身是一个 web 交互库, 这里说的是它里面的 核心 Diff 算法
据说灵感源自 两个不同文本之间的 Diff 中的 去除相同的 前后缀
```js
// 强大的指针们
let j = 0 // j 为指向新旧 children 中第一个节点的索引
let prevEnd = prevChildren.length - 1 // 指向 旧 children 最后一个节点的索引
let nextEnd = nextChildren.length - 1 // 指向 新 children 最后一个节点的索引

outer: {
  // 更新 ("去掉") 相同的前缀节点
  let prevVNode = prevChildren[j]
  let nextVNode = nextChildren[j]
  // while 循环向后遍历，直到遇到拥有不同 key 值的节点为止
  while (prevVNode.key === nextVNode.key) {
    // 调用 patch 函数更新
    patch(prevVNode, nextVNode, container)
    j++ // 前指针向后移动
    if (j > prevEnd || j > nextEnd) {
      // 这个时候说明 新or旧两个 children 至少有一个已经全部参与了 patch
      // 没必要再继续 while 了, 使用 js label 语句进行 break 
      break outer
    }
    prevVNode = prevChildren[j]
    nextVNode = nextChildren[j]
  }

  // 更新 ("去掉") 相同的后缀节点

  prevVNode = prevChildren[prevEnd]
  nextVNode = nextChildren[nextEnd]

  // while 循环向前遍历，直到遇到拥有不同 key 值的节点为止
  while (prevVNode.key === nextVNode.key) {
    // 调用 patch 函数更新
    patch(prevVNode, nextVNode, container)
    prevEnd-- // 旧 children 指针向前移动
    nextEnd-- // 新 children 指针向前移动
    if (j > prevEnd || j > nextEnd) {
      break outer
    }
    prevVNode = prevChildren[prevEnd]
    nextVNode = nextChildren[nextEnd]
  }
}

// 两轮 while 结束之后, 根据 j prevEnd nextEnd 之间的关系来决定节点 "多退" 还是 "少补"
if (j > prevEnd && j <= nextEnd) { // 说明 nextChildren 从 j -> nextEnd 之间的节点应作为新节点插入
  // 所有新节点应该插入到位于 nextPos 位置的节点的前面
  const nextPos = nextEnd + 1
  const refNode = nextPos < nextChildren.length ? nextChildren[nextPos].el : null
  // 采用 while 循环，调用 mount 函数挂载节点
  while (j <= nextEnd) {
    container.insertBefore(nextChildren[j++].el, refNode)
  }
} else if (j > nextEnd) { // 说明 prevChildren 从 j -> prevEnd 之间的节点应该被移除
  while (j <= prevEnd) {
    container.removeChild(prevChildren[j++].el)
  }
} else { // 意味着现在开始需要移动 DOM
  const nextLeft = nextEnd - j + 1 // 新 children 中剩余未被处理节点的 数量
  const source = [] // 构建一个数组用于计算一个最长递增字序列
  for (let i = 0; i < nextLeft; ++i) { // 初始化这个数组
    source.push(-1) // 初始 -1 后存入新 children 中的节点在 旧 children 中的位置
  }
  // 下面这部分有点像 react 的核心 diff 算法
  const prevStart = j
  const nextStart = j
  let moved = false
  let pos = 0
  // 构建索引表 (可以先把 key 对应的索引缓存下来, 避免双层循环去找)
  const keyIndex = {}
  for(let i = nextStart; i <= nextEnd; i++) {
    keyIndex[nextChildren[i].key] = i
  }
  // 遍历旧 children 的剩余未处理节点
  let patched = 0 // 代表目前旧节点中 已经更新过的节点的 数量
  for(let i = prevStart; i <= prevEnd; i++) {
    prevVNode = prevChildren[i]

    if (patched < nextLeft) { // 只要还没到 剩余应更新数量 说明不是多余节点
      // 通过索引表快速找到新 children 中具有相同 key 的节点的位置
      const k = keyIndex[prevVNode.key]

      if (typeof k !== 'undefined') {
        nextVNode = nextChildren[k]
        // patch 更新
        patch(prevVNode, nextVNode, container)
        // 更新 source 数组
        source[k - nextStart] = i
        // 判断是否需要移动
        if (k < pos) {
          moved = true
        } else {
          pos = k
        }
      } else { // 没找到
        // 说明旧节点在新 children 中已经不存在了, 应该移除
        container.removeChild(prevVNode.el)
      }
    } else { // 说明是旧 children 中多余出来的节点, 应该移除
      container.removeChild(prevVNode.el)
    }
  }
  // 开始移动 DOM
  if (moved) {
    const seq = lis(sources) // 计算最长递增子序列
    let j = seq.length - 1 // 指向最长递增子序列中最后一个位置
    for (let i = nextLeft; i >= 0; --i) { // 从剩余未处理节点最后一个位置往前            
      if (source[i] === -1 || i !== seq[j]) { // 说明该节点是新的需要挂载 或 说明该节点需要移动
        // 该节点在新 children 中的真实位置索引
        const pos = i + nextStart
        const nextVNode = nextChildren[pos]
        // 该节点下一个节点的位置索引
        const refNode = pos + 1 < nextChildren.length
            ? nextChildren[pos + 1].el
            : null
        // 移动 DOM
        container.insertBefore(nextVNode.el, refNode)
      } else { // 而当 i === seq[j] 时，说明该位置的节点不需要移动, 并让 j 指向下一个位置
        j--
      }
    }
  }
}

// 最长递增子序列 是一个原集合的子集 (可以用位置索引表示)
// 1. 它里面的每个元素 (索引) 指向的原数组的元素是递增的
// 2. 并在原集合中位置顺序不变
// 3. 但是可以是不连续的

function lis (arr) {
  // 首先得到一个数组
  // 里面每个元素代表 原数组中从它的下标元素为起点 到原数组末尾的 最长递增子序列 的长度
  let lis = Array.from(arr, x => 1)
  let max = lis[0] // 存储当前最长子序列 起始点 的 下标
  for (let i = arr.length - 2; i >= 0; --i) { // 可以从倒数第二位开始往前
    let j = i, count = lis[i]
    while (++j < arr.length) {
      if (arr[i] < arr[j]) {
        count = Math.max(lis[i] + lis[j], count)
      }
    }
    lis[i] = count
    if (count > lis[max]) {
      max = i
    }
  }

  // 再从 max 开始记录一个最终结果
  // 即 最长递增子序列中具体每个元素 在原数组中的位置
  let res = []
  for (let i = 0; i < lis[max]; ++i) {
    for (let j = max + i; j < arr.length; ++j) {
      if (lis[j] === lis[i] - i) {
        res.push(j)
        break
      }
    }
  }
  return res
}
```
总结 inferno diff 算法的快 应该来自:
1. 预处理可以去除前后缀, 并做了一部分的新旧 children 多退少补;
2. 判断移动 DOM 阶段将 O(n^2) 的复杂度扁平化, 做到 O(n) 的时间复杂度;
3. 移动 DOM 阶段使用了计算最长递增子序列的方式最少化需要移动的 DOM 数量;

## 跨平台 render
不足之处
上述的算法中, 使用的平台相关 API 全部是 Web 平台的 API, 比如
```js
container.removeChild()
container.insertBefore()
```
在 组件的本质 中就提到过, 虚拟 DOM 的核心价值, 是提供了 跨平台渲染 的能力
实际上, VNode 这个东西的类型可以是任意的, 对它操作的行为不应受任何真实平台约束
这就需要封装一些专用的函数比如: removeVNode, 它可以判断一个 VNode 的类型, 并采用合适的方式移除它所渲染的 真实 DOM
再比如, 需要移除的 VNode 描述的是一个特定平台组件, 当然也需要调用该平台的 beforeUnmount 和 unmounted 等等 API
所以需要对目前的渲染器进行进一步抽象层封装

列举 Web 浏览器所提供的常用 DOM API:
 - document.querySelector 查找元素节点
 - document.createElement 创建标签元素
 - document.createTextNode 创建文本元素
 - el.nodeValue = '' 修改文本元素内容
 - el.className = '' 修改元素的类名
 - el.setAttribute 设置元素属性
 - el.removeCHild 移除 DOM 元素
 - el.insertBefore 插入 DOM 元素
 - el.appendChild 获取当前元素的父元素
 - el.nextSibling 获取当前元素的下一个兄弟元素
 - ...
以 DOM API 为范本总结, 跨平台渲染器至少应该允许各平台自定义的部分为:
  1. 可以自定义元素的新建, 删除, 查找等操作
  2. 可以自定义元素自己的 props & attrs 的修改操作
根据前文, 一个渲染器包含 mount 和 patch 两个阶段, 这两个阶段中分别调用具体平台的 API 来完成真实渲染
可以将创建基于不同平台的 API 的渲染器的工作封装到一个工厂函数中:
```js
export default function createRenderer(options) {
  const {
    nodeOps: {
      createElement: platformCreateElement,
      createText: platformCreateText,
      setText: platformSetText, // 等价于 Web 平台的 el.nodeValue
      appendChild: platformAppendChild,
      insertBefore: platformInsertBefore,
      removeChild: platformRemoveChild,
      parentNode: platformParentNode,
      nextSibling: platformNextSibling,
      querySelector: platformQuerySelector
    },
    patchData: platformPatchData
  } = options

  function render (vnode, container) { /* ...do mount ...do patch */ }
  function mount (vnode , container) { /* ...使用 options.api 进行 mount */ }
  function mountElement(vnode, container) {
    const el = platformCreateElement(vnode, container)
    // 省略...
  }
  // 省略...
  function patch (prevVNode, nextVNode, container) { /* ...使用 options.api 进行 patch */ }
  // 省略...

  return { render }
}
```
责任重大的 options 需要统一设计:
```js
const { render } = createRenderer({
  nodeOps: { // 自定义元素的新建, 删除, 查找等操作的函数集合
    createElement() { /* ... */ },
    createTextNode() { /* ... */ },
    removeChild(parent, child) {
      // 差不多像这样
      parent.removeChild(child)
    }
    // 更多操作 ...
  },
  patchData (el, key, oldValue, newValue) {
    // 这个函数其实就是在前面代码 mountElement 和 patchElement 中对各种属性的那一堆操作的封装
    switch (key) {
    case 'style':
      for (let k in nextValue) {
        el.style[k] = nextValue[k]
      }
      for (let k in prevValue) {
        if (!nextValue.hasOwnProperty(k)) {
          el.style[k] = ''
        }
      }
      break
    case 'class':
      el.className = nextValue
      break
    default:
      if (key[0] === 'o' && key[1] === 'n') {
        // 事件
        // 移除旧事件
        if (prevValue) {
          el.removeEventListener(key.slice(2), prevValue)
        }
        // 添加新事件
        if (nextValue) {
          el.addEventListener(key.slice(2), nextValue)
        }
      } else if (/\[A-Z]|^(?:value|checked|selected|muted)$/.test(key)) {
        // 当做 DOM Prop 处理
        el[key] = nextValue
      } else {
        // 当做 Attr 处理
        el.setAttribute(key, nextValue)
      }
      break
    }
  }
})
```
实现了所有的 nodeOps 接口函数 和 patchData 函数之后, 就完成了某个平台下的渲染器

跨平台 render 的应用:
  - @vue/runtime-test
    在没有真实 runtime 的情况下假装有一个 runtime:
    ```js
    const { render } = createRenderer({
      nodeOps: {
        createElement(tag) {
          const customElement = {
            type: 'ELEMENT',
            tag
          }
          console.table({
            type: 'CREATE ELEMENT',
            targetNode: customElement
          })
          return customElement
        },
        // more ...
      }
    })
    ```
    这样就能模拟实现一个真实 DOM 对象了, 可以用来模拟运行时做测试
  - 渲染到 pdf canvas 文件系统等
    我们可以实现一个特定平台自定义渲染器, 它能够将 Vue 组件渲染为 各种文件

## 异步渲染

# 疑问
关于 双端比较 diff 算法比第一种 diff 算法的优势之处
理解的比较笼统, 还是没有想得太清楚, 需要写个实例来测试一下

# 参考
http://hcysun.me/vue-design/zh
https://zhuanlan.zhihu.com/p/150732926
https://neil.fraser.name/writing/diff/
https://juejin.cn/post/6934698604955172877

# 补充
* Vue 为什么不需要时间分片?
https://juejin.cn/post/6844904134945030151
https://zhuanlan.zhihu.com/p/133819602

* 虚拟 DOM 与 React Fiber
https://www.zhihu.com/question/31809713
https://www.zhihu.com/question/31809713
https://zhuanlan.zhihu.com/p/26027085
https://www.cnblogs.com/WindrunnerMax/p/14781432.html
https://www.jb51.net/article/209310.htm

* 其他
https://segmentfault.com/a/1190000019292569

# 总结
  组件 = 
  响应式数据 (Object.defineproperty, Proxy) + 
  render 函数 (产出虚拟 DOM) + 
  模板编译 (产出 render 函数) + 
  逻辑复用 (mixin, composive api, hooks, 高阶函数)
