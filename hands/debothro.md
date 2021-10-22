# 防抖 (debounce)
名字起得很他妈不直观, 防抖到底啥意思?
防抖的意思就是: 多次触发事件, 但我一定在 (最后一次) 事件触发的 n 秒后执行
即 如果是在上次执行后 第一次触发的 n 秒内再次触发, 则以新的事件时间为准再次等 n 秒后执行
感觉上就是, 你点, 你继续点, 等你冷静了, 我再动

而 debounce 函数, 一般是一个利用了闭包的高阶函数:
```js
function debounce(fn, wait) {
  let timeout
  return function (...args) {
    timeout && clearTimout(timeout)
    timeout = setTimeout(fn.bind(this, ...args), wait)
  }
}
```

# 节流 (throttle)
节流和防抖解决的都是同一个问题, 只不过效果不同
节流是按第一次触发执行后 等待 n 秒, 然后的触发才可再次执行
感觉上就是, 你点了, n 秒之内再点无效, n 秒之后再来

throttle 函数也是一个高阶函数:
```js
function throttle(fn, wait) {
  let prev = 0
  return function (...args) {
    let now = Date.now()
    if (now - prev > wait) {
      fn.apply(this, args)
      prev = now
    }
  }
}
```

# 参考
https://github.com/mqyqingfeng/Blog/issues/22
https://github.com/mqyqingfeng/Blog/issues/26