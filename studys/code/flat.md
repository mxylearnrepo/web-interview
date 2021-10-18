[面试官连环追问：数组拍平（扁平化） flat 方法实现](https://segmentfault.com/a/1190000021366004)

```js
const flat = arr => arr.reduce((pre, cur) => {
  return pre.concat(Array.isArray(cur) ? flat(cur) : cur)
}, [])
```