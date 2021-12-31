# .onxxx 与 .addEventListener('xxx', handler, useCapture) 的区别?
首先 a 是每次赋值 都覆盖前一次的函数, 也就是只能响应一个函数
  并且只能通过冒泡的方式执行
而 b 则每次 add 都是增加, 事件触发时按 add 顺序 (一般是这样) 执行
  并且可以设置 useCapture 为 true 来在捕获阶段执行 (默认是冒泡 false)

# 事件委托
e.currentTarget 与 e.target 在最终目标处一样, 在冒泡和捕获阶段不一样
事件委托就在捕获阶段 发现了 target 是某个预设的元素, 就直接执行对应 handler 逻辑
对于事件目标元素有很多的情况 (比如给一堆 li 添加点击), 事件委托可以提升性能
对于目标元素在绑定事件时可能还未产生的情况, 使用事件委托可以做到提前绑定

# 自定义事件
```js
const patchMethod = type =>
  () => {
    const result = window.history[type].apply(this, arguments)
    // 添上一个原来没有的事件
    const event = new Event(type)
    event.arguments = arguments
    window.dispatchEvent(event)
    return result
  }
history.pushState = patchMethod('pushState')
history.replaceState = patchMethod('replaceState')
window.addEventListener('replaceState', e => {
  // e.arguments
})
```