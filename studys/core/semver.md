# 比较版本号大小
```js
function compare(v1, v2) {
  let i = 0
  const arr1 = v1.split('.')
  const arr2 = v2.split('.')
  
  while(true) {
    const c1 = arr1[i]
    const c2 = arr2[i]
    i++
    if (c1 === undefined || c2 === undefined) {
      return arr1.length - arr2.length
    }

    if (c1 === c2) continue

    return c1 - c2 === NaN ? c1.charCodeAt() - c2.charCodeAt() : c1 - c2
  }
}
```
