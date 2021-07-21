# Fetch 规范
有别于 Ajax 规范的另一种 浏览器端 发送请求方式

XMLHTTPRequest 存在一些缺点:
  配置和使用较为繁琐(见前文)
  基于事件的异步模型不够友好(为啥?)

Fetch 的推出 (es6) 主要为了解决上述问题:
```js
fetch('http://xxx')
  .then(function (res) {
    console.log(res)
  })
  .catch(function (err) {
    console.error("ERROR!", err)
  })
```
非常简洁

