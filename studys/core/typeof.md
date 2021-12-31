typeof 关键字可以区分出 undefined Boolean Number String Symbol Function 和 Object
对于属于 Object 的数据, 没法再区分了, 比如 null, Date, Array 等, 这种一般通过 instanceof 去匹配
但是我们可以实现一个基于 Object.prototype.toString 的更屌的 typeof:
```js
function typeof(obj) {
  return Object.prototype.toString.call(obj).slice(8, -1).toLowerCase()
}
```
