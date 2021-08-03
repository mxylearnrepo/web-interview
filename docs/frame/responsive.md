# 响应式原理
MVVM: Model-View-ViewModel
数据与视图之间没有直接的关系, vm层负责它们的交互.
vm就是一个同步 view 和 model 的 对象, 双向绑定装置.
vm会 watch 数据变化, 然后更新dom.
参考: 
  https://www.jianshu.com/p/26e24ce84c4b
  https://www.jianshu.com/p/8cde476238f0

观察者模式与发布订阅模式的区别是: 观察者模式没有事件中心, 观察者需要知道订阅者的存在

模拟 Vue 响应式实现 (观察者模式):
Vue -> Observer 劫持数据 -> Dep 发布者 -> Watcher 观察者    
Vue -> Compiler 解析指令 -> Watcher 观察者 

Vue 类:
```js
// 1. 负责接收初始化参数
// 2. 负责把 data 中的数据注入到 vue 实例 (this 上), 并转化成 getter/setter
// 3. 负责调用 observer 监听 data 中所有数据变化
// 4. 负责调用 compiler 解析指令/差值表达式 (操作 dom)
export default class Vue {
  constructor (options) {
    this.$options = options || {}
    this.$data = options.data || {}
    this.$el = typeof options.el === 'string' ? document.querySelector(options.el) : options.el

    this._proxyData(this.$data) // 立即注入 data

    new Observer(this.$data) // 立即转化 data

    new Compiler(this) // 立即解析 dom
  }

  _proxyData (data) { // 让 vue 代理 data 中的属性
    // 遍历 data 中的所有属性
    Object.keys(data).forEach(key => {
      Object.defineProperty(this, key, {
        enumberable: true,
        configurable: true,
        get () {
          return data[key]
        },
        set (newValue) {
          if (newValue === data[key]) return
          data[key] = newValue
        }
      })
    })
  }
}
```
Observer:
```js
// 1. 负责递归的把 data 中的属性转成响应式数据
// 2. 数据变化时发送通知给 Dep 发布者
export default class Observer {
  constructor (data) {
    this.walk(data)
  }

  walk (data) { // 遍历 data 对象
    // 判断 data 是否是对象
    if (!data || typeof data !== 'object') {
      return 
    }

    // 遍历 data 所有属性
    Object.keys(data).forEach(key => {
      this.defineReactive(data, key, data[key])
      this.walk(data[key]) // 递归
    })
  }

  defineReactive (obj, key, val) { // 把被遍历的对象变成响应式
    const that = this
    const dep = new Dep()

    Object.defineProperty(obj, key, {
        enumberable: true,
        configurable: true,
        get () { // 如果不是传入的 val 而是使用 obj[key] 会出现循环 get 导致内存函数调用栈溢出
          Dep.target && dep.addSub(Dep.target) // 收集依赖
          return val // 这时候这个 val 存在于一个闭包内, 并不会被释放
        },
        set (newValue) {
          if (newValue === val) return
          val = newValue // val 不会被释放, 故而当 set 修改它, get 得到的还是它, 所以: 自给自足了
          // 那么 data[key] 里面的东西其实一直是初始的那个值...?
          that.walk(newValue) // in case of newValue 是个对象
          // 监听到了数据变化, 发送通知告诉 Dep: 触发依赖
          dep.notify() // 同样地, 这个 dep 实例也被使用在了一个闭包内
        }
    })
  }
}
```
Compiler:
```js
// 1. 负责编译模板 template -> render 函数, 在这个过程中解析指令/差值表达式
// 2. 负责页面的首次渲染
// 3. 负责 data 数据变化后得重新渲染页面
export default class Compiler {
  constructor (vm) {
    this.el = vm.$el
    this.vm = vm
    compile(this.el)
  }

  compile (el) { // 遍历 el 下面所有节点, 编译模板
    let childnodes = el.childNodes // 一个伪数组
    Array.from(childnodes).forEach(node => {
      if (this.isTextNode(node)) {
        this.compileText(node)
      } else if (this.isElementNode(node)) {
        this.compileElement(node)
      }

      // 判断 node 是否还有子节点, 如果有就递归调用 compile
      if (node.childNodes && node.childNode.length) {
        this.compile(node)
      }
    })
  }

  compileElement (node) { // 编译元素节点, 处理指令
    Array.from(node.attributes).forEach(attr => {
      let attrName = attr.name
      if (this.isDirective(attrName)) {
        attrName = attrName.substr(2) // 去掉了 v- 两个字符
        let key = attr.value // v-xx="abc" 的 abc
        this.update(node, key, attrName)
      }
    })
  }

  update (node, key, attrName) { // 找到指令对应的 updater
    let updateFn = this[attrName + 'Updater']
    updateFn && updateFn.call(this, node, this.vm[key], key)
  }

  textUpdater (node, val, key) { // 处理 v-text 指令
    node.textContent = val
    new Watcher(this.vm, key, newValue => {
      node.textContent = newValue
    })
  }

  modelUpdater (node, val, key) { // 处理 v-model 指令
    node.value = val
    new Watcher(this.vm, key, newValue => {
      node.value = newValue
    })
    // 双向绑定
    node.addEventListener('input', () => {
      this.vm[key] = node.value
    })
  }

  // 如果需要处理 v-if 等其他一些指令, 只需要在这继续添加 xxxUpdater 就可以了
  // ? 如何处理自定义指令 ?

  compileText (node) { // 编译文本节点, 处理差值表达式
    // 差值表达式长这样: {{ val }} 
    // 正则表达式:
    let reg = /\{\{(.+?)\}\}/
    let val = node.textContent
    if (reg.test(val)) {
      let key = RegExp.$1.trim() // 去掉 val 前后的空格
      node.textContent = val.replace(reg, this.vm[key]) // 这里为什么要 replace ??
      // 创建 watcher 对象, 当数据改变更新视图
      new Watcher(this.vm, key, newValue => {
        node.textContent = newValue // 因为 set 还未执行完 所以不能 this.vm[key]
      })
    }
  }

  isDirective (attrName) { // 判断是否是指令
    return attrName.startsWith('v-')
  }

  isTextNode (node) { // 判断是文本节点
    return node.nodeType === 3
  }

  isElementNode (node) { // 判断是元素节点
    return node.nodeType === 1
  }
}
```
Dep (发布者):
```js
// 1. 负责收集依赖, 添加观察者 (watcher)
// 2. 负责通知所有观察者
export default class Dep {
  constructor () {
    this.subs = [] // 存储所有的 watcher
  }

  addSub (sub) { // 添加 watcher
    // 判断 sub 存在且是不是一个 watcher
    if (sub && sub.update) {
      this.subs.push(sub)
    }
  }

  notify () { // 当数据发生变化时会被调用, 在这里面再通知所有的 watcher
    // 遍历 subs 调用每一个的 update 方法
    this.subs.forEach(sub => {
      sub.update()
    })
  }
}
```
Watcher:
```js
// 1. 负责当数据变化时响应这种变化, dep 通知其所有的 watcher 实时更新
// 2. 自身实例化的同时, 往 dep 对象中添加自己
export default class Watcher {
  constructor (vm, key, cb) {
    this.vm = vm // vue 实例
    this.key // watch 的数据属性名称
    this.cb // 不同 watcher 所做的具体事情, 当数据变化的时候调用

    // 把当前这个 watcher 对象记录到 Dep 类的静态属性 target 中,
    Dep.target = this

    // 会立即触发一次该属性的 get 方法, 在 get 方法中调用了 addSub
    this.oldValue = vm[key] // 用于比较新旧值有没有发生变化, 没有则不要触发 set

    // 立即清理这个 Dep.target, 防止什么地方又访问 vm[key] 造成同一个 watcher 重复添加
    Dep.target = null
  }

  update () { // 当数据发生变化时调用 cb
    let newValue = this.vm[this.key]
    if (newValue === this.oldValue) {
      return 
    }
    this.oldValue = newValue // 这里为什么不需要这个呢?
    this.cb(newVlaue)
  }
}
```

### 在模拟 Vue.js 响应式源码的基础上实现 v-html 指令，以及 v-on 指令
```js
// in Compiler class ...
update (node, key, attrName) { // 找到指令对应的 updater
  attrName = attrName.split(':')
  let updateFn = this[attrName[0] + 'Updater']
  if (attrName.length > 1) {
    updateFn && updateFn.call(this, node, this.vm[key], key, attrName[1])
  } else {
    updateFn && updateFn.call(this, node, this.vm[key], key)
  }
}

htmlUpdater (node, val, key) {
  node.innerHTML = val
  new Watcher(this.vm, key, newValue => {
    node.innerHTML = newValue
  })
}

onUpdater (node, val, key, eventName) {
  const fn = val.bind(this.vm)
  node.addEventListener(eventName, fn, false)
  new Watcher(this.vm, key, newValue => {
    node.removeEventListener(eventName, fn, false)
    node.addEventListener(eventName, newValue.bind(this.vm), false)
  })
}
```
