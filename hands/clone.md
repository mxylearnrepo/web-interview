[如何写出一个惊艳面试官的深拷贝](https://cloud.tencent.com/developer/article/1497418)
```js
function clone(target, map = new WeakMap()) {
  if(typeof target === 'object') {
    let cloneObj = Array.isArray(target) ? [] : {} // 处理数组
    if (map.get(target)) { // 处理循环引用
      return target
    }
    map.set(target, true)
    for (const key in target) {
      cloneObj[key] = clone(target[key])
    }
    return cloneTarget
  } else {
    return target
  }
}
```