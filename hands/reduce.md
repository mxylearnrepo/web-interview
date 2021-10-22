# reduce
reduce 本意: 减少, 缩小, 使还原, 使变弱
MDN 的解释: reduce 方法通过一个给定的执行器函数, 和初始值, 对数组从前到后每一项进行函数加工, 最终产生一个结果
```js
arr.reduce((preVal, currVal, currIndex, arr), initialValue)
```
initialValue 如果没传就默认是 arr 第一个元素

就相当于是包裹了一个对 arr 的遍历, 实现
```js
Array.prototype.reduce = Array.prototype.reduce || function (func, initialValue) {
  var arr = this
  var current = typeof initialValue === undefined ? arr[0] : initialValue
  var startPoint = typeof initialValue === 'undefined' ? 1 : 0
  arr.slice(current).forEach((val, index) => {
    current = func(current, val, index + startPoint, arr)
  })
  return current
}
```

# only
```js
var only = function (target, keys) {
  obj = obj || {}
  keys = Array.isArray(keys) ? keys : []
  return keys.reduce(function(ret, key) {
    if (!target[key]) {
      return ret
    }
    ret[key] = target[key]
    return ret
  }, {})
}
```

# pipe 
```js
const pipe = (...fns) => input => functions.reduce(
  (ret, fn) => fn(ret), input
)
```

# flat
```js
const flat = arr => arr.reduce((pre, cur) => {
  return pre.concat(Array.isArray(cur) ? flat(cur) : cur)
}, [])
```
[更多实现方式](https://segmentfault.com/a/1190000021366004)
