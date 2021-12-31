# 响应式基本原理
## 什么是 MVVM ?
Model-View-ViewModel
操作数据, 就是操作视图, 就是操作 DOM (所以无须直接操作 DOM)

将 MVVM 整个过程串起来就是:
  首先对 **所有的数据 (model)** 进行拦截 (Object.defineProperty 或 Proxy)
    在拦截器里面 **递归的** 对每一个属性的 getter 和 setter 进行加工
  当访问到一个数据的时候, 就会触发它对应的 getter, 在这个 getter 中会记录这个**依赖的 Watcher**
  当修改了一个数据的时候, 在拦截器里它对应的 setter 中会通知 Watcher 这个变更
  最终在 Watcher 里面调用注册的回调, 进行了 View 的更新 (update)

负责做整个这件事的模块, 就叫 ViewModel, 可见 ViewModel 是一个双重发布/订阅模式

## 发布/订阅模式
[参考](./event_bus.md)

## Object.defineProperty 不能监听数组
这种说法指的是不能对数组的常用 APIs 进行监听
主流框架的解决办法是对这些方法进行重写: 
```js
const arrExtend = Object.create(Array.prototype)
const arrMethods = [
  'push', 'pop', 'shift', 'unshift', 'splice', 'sort', 'reverse'
]

arrMethods.forEach(method => {
  const oldMethod = Array.prototype[method]
  const newMethod = function (...args) {
    oldMethod.apply(this, args)
    // ... 我们可以在这里执行一次 setter 中同样的逻辑
  }
  arrExtend[method] = newMethod // 为什么需要一个 arrExtend?
})
Array.prototype = Object.assign(Array.prototype, arrExtend) // 为什么要重新赋值?
```
这种给内置方法打补丁的做法其实**不是很安全, 也不优雅**, 更好的实现方式是 Proxy

## ES6 Proxy
Proxy 与 Object.defineProperty 的区别是:
  defineP 不能监听数组方法, 需要对方法重写; Proxy 可以监听数组 APIs, 因为数组也是一个对象
  defineP 必须遍历对象的每个属性进行监听; Proxy 是监听整个对象, 不需要遍历对象的每个属性
  对于对象里有的属性值又是对象的这种**深层结构**, 无论是 defineP 还是 Proxy 都需要进行递归处理

## 双向绑定
我以为双向绑定其实与 MVVM 无关, 但是有些面试官可能会混为一谈, 那没办法
```js
// 假设 node 是一个 HtmlNode, 也可以用 zepto
Array.from(node.attributes).forEach(attr => {
  const name = attr.name
  const key = attr.value
  if (name.includes('v-')) {
    node.value = data[key] // data 是我们的响应式数据
    node.addEventListener('input', e => {
      const newValue = e.target.value // === e.currentTarget.value
      data[key] = newValue // 触发了 setter
    })
  }
})
```
