[如何写出一个深拷贝](https://cloud.tencent.com/developer/article/1497418)

# 深拷贝 (deep clone)
```js
function clone(target, map = new WeakMap()) {
  if (map.get(target)) { // 处理循环引用
    return target
  }

  // 获取当前值的构造函数：获取它的类型
  let constructor = target.constructor
  // 检测当前对象target是否与正则、日期格式对象匹配
  if (/^(RegExp|Date)$/i.test(constructor.name)) {
    // 创建一个新的特殊对象(正则类/日期类)的实例
    return new constructor(target)
  }

  if ((typeof target === "object" || typeof target === "function") && target !== null) {
    map.set(target, true)
    const cloneObj = Array.isArray(target) ? [] : {} // 处理数组
    for (const key in target) {
      cloneObj[key] = clone(target[key], map)
    }
    return cloneTarget
  } else {
    return target
  }
}
```

# 实现 Object.assign
Object.assign 是浅拷贝
```js
const assign = function(target, ...source) {
  if (target == null) {
    throw new TypeError('Cannot convert undefined or null to object')
  }
  let ret = Object(target)
  source.forEach(function(obj) {
    if (obj != null) {
      for (let key in obj) {
        if (obj.hasOwnProperty(key)) { // 为啥多此一举判断?
          ret[key] = obj[key]
        }
      }
    }
  })
  return ret
}
```
