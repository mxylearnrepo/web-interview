[一文搞懂V8引擎的垃圾回收](https://juejin.cn/post/6844904016325902344)

# 原因
首先就是 V8 引擎对内存的使用存在大小限制, 原因是 Js 单线程, 如果垃圾回收太耗时就会阻塞主线程
这个限制在 nodejs 环境中可以通过 --v8-options 进行配置

# 策略
首先划分内存, 主要是新生代区和老生代区
新生代区: 
  以空间换时间, 分为两半 from 和 to 区域
  新产生的对象进入 from, 在触发下一回收阶段时, 把存在强引用的对象拷贝到 to
  删除 from 区所有对象, 这时 from 和 to 身份互换, to 作为下一次的 from
  拷贝到 to 区的对象如果满足条件 (存活了很多轮), 可以晋升到 老生代区 享福
老生代区:
  比新生代区大, 管理大量存活对象, 所以不能使用新生代牺牲空间的方式进行 GC
  所以使用了 标记清除 和 标记整理算法, 具体算法暂未研究

# Javascript 的内存管理
Js 的内存空间可以分为栈空间和堆空间, 上面的 GC 策略发生在 堆空间
  栈空间: 由操作系统 (OS) 自动分配和释放, 存放函数调用相关的数据 (参数值, 局部变量值等)
  堆空间: 一般需要由开发者分配和释放, 这部分空间需要考虑垃圾回收问题
通常情况下, 基本数据类型 (undefined, null, string, boolen, number 等) 的值可以直接保存在栈空间中, 占有固定的内存空间
而引用类型 (Object, Array, Function 等) 占用的内存大小不固定, 保存在堆空间中, 需要在栈中保存它们的引用来访问

# 如何避免内存泄露
尽可能减少全局变量
```js
// element 是一个全局变量
var element = document.getElementById("element")
function remove() {
  element.parentNode.removeChild(element)
  // 虽然 DOM 节点的 element 对象被移除了, element 对象还在
  // 需要加上 element = ''
}
```

记得手动清除定时器
```js
function foo() {
  var name = 'maos'
  setInterval(function() {
    console.log(name)
  }, 1000)
}
foo()
// name 因为 setInterval 形成闭包无法释放
// 需要 clearInterval
```

记得清除 DOM 引用
```js
var element = document.getElementById("element")
element.innerHTML = '<button id="button">点击</button>'
var button = document.getElementById("button")
button.addEventListener('click', () => {})
element.innerHTML = ''
// button 和 button 的 DOM 节点变量都依然存在
// 需要 button.removeEventListerner 和 button = ''
```

使用闭包时谨慎小心
```js
function foo() {
  let name = 'maos'
  function bar() {
    console.log(name)
  }
  return bar
}
let bar = foo()
bar() // name 永远不会被释放
```

使用 weakMap 和 weakSet
