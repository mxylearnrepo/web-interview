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
  var current = typeof initialValue === 'undefined' ? arr[0] : initialValue
  var startPoint = typeof initialValue === 'undefined' ? 1 : 0
  arr.slice(startPoint).forEach((val, index) => {
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
pipe 把传入的 fns 从前往后套娃执行
```js
const pipe = (...fns) => input => fns.reduce((pre, cur) => cur(pre), input)
```
而 compose 是 [pipe](./reduce.md) 的一个反向
```js
const compose = (...fns) => fns.reduce((pre, cur) => (...args) => pre(cur(...args)))

```
上头 pipe 和 compose 的实现没有做这俩个判断
```js
if (!fns.length) return v => v
if (fns.length === 1) return fns[0]
```

# flat
数组扁平化
```js
const res = arr.flat(Infinity)
```
最直观的实现是用递归 (类似深拷贝)
```js
const flat = arr => {
  const res = []
  for (let item of res) {
    if (Array.isArray(item)) {
      res = res.concat(flat(item))
    } else {
      res.push(item)
    }
  }
  return res
}
```
然后用 reduce 实现
```js
const flat = arr => arr.reduce((pre, cur) => {
  return pre.concat(Array.isArray(cur) ? flat(cur) : cur)
}, [])
```
[更多实现方式](https://segmentfault.com/a/1190000021366004)
