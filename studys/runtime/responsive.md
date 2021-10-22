# 响应式基本原理
## 什么是 MVVM ?
Model-View-ViewModel
一句话总结 Web 前端 MVVM: 操作数据, 就是操作视图, 就是操作 DOM (所以无须直接操作 DOM)
将 MVVM 整个过程串起来就是:
  首先对 **所有的数据 (model)** 进行拦截(Object.defineProperty) 或代理 (Proxy), **递归的**对每一个属性的 getter 和 setter 进行加工
  当访问到 **依赖的数据** 的时候 (可能发生在模板编译等地方) 就会触发这个数据刚才添加的 getter, 在这个 getter 中会记录这个**依赖的 Watcher**, 同时在它的 setter 中会通知 Watcher 这个变更
  最终在 Watcher 里面进行了 View 的更新
负责做整个这件事的模块, 就叫 ViewModel
可见 ViewModel 是一个双重发布/订阅模式

## 发布/订阅模式
JavaScript 是一个事件驱动型语言, 事件驱动就是发布/订阅模式
这个模式的好处是对代码进行解耦, 实现"高内聚, 低耦合"
一个简单例子:
```js
class EventEmitter {
  constructor() {
    this.eventsmap = {}
  }
  on(name, handler) {
    if (!this.eventsmap[name]) {
      this.eventsmap[name] = []
    }
    this.eventsmap[name].push(handler)
  }
  emit(name) {
    this.eventsmap[name].length && this.eventsmap[name].forEach(handler => handler())
  }
}
```

## Object.defineProperty 不能监听数组
这种指的是不能对数组的常用 APIs 进行监听
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
这种给内置方法打补丁的做法其实不是很安全, 也不优雅, 更好的实现方式是 Proxy

## 数据拦截与代理
Proxy 与 Object.defineProperty 的区别是:
  define 不能监听数组方法, 需要对方法重写; Proxy 可以监听数组, 因为数组也是一个对象
  define 必须遍历对象的每个属性进行监听; Proxy 是监听整个对象, 不需要遍历对象的每个属性
  对于属性中有对象的深层结构, 无论是 define 还是 Proxy 都需要进行递归处理

