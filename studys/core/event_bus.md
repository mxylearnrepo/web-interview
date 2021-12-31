# 事件总线 (发布订阅模式)
JavaScript 是一个事件驱动型语言, 事件驱动就是发布/订阅模式
```js
class EventEmitter {
  constructor() {
    this.cache = {}

    on(name, fn) {
      if (this.cache[name]) {
        this.cache[name].push(fn)
      } else {
        this.cache[name] = [fn]
      }
    }

    off(name, fn) {
      let tasks = this.cache[name]
      if (tasks) {
        const index = tasks.findIndex(f => f === fn || f.callback === fn)
        if (index >= 0) {
          tasks.splice(index, 1)
        }
      }
    }

    emit(name, ...args) {
      if (this.cache[name]) {
        // 为什么创建一个副本, 因为如果某个回调内继续注册相同 name 的 on, 可能会造成死循环
        let tasks = this.cache[name].slice()
        for (let task of tasks) {
          fn(...args)
        }
      }
    }

    once(name, ...args) {
      this.emit(name, ...args)
      delete this.cache[name]
    }
  }
}
```
这个模式的好处是对代码进行解耦, 实现"高内聚, 低耦合"

## 关于 EventBus vs Vuex
[EventBus & Vuex?](https://juejin.cn/post/6844903733256519694)
这俩都是**集中**状态管理方案, 一个心智负担轻, 一个重但更适合大型复杂项目
