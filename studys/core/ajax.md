# AJAX
```js
const getJSON = function(url) {
  return new Promise((resolve, reject) => {
    const xhr = XMLHttpRequest ? new XMLHttpRequest() : new ActiveXObject('Mscrosoft.XMLHttp')
    xhr.open('GET', url, false)
    xhr.setRequestHeader('Accept', 'application/json')
    xhr.onreadystatechange = function() {
      if (xhr.readyState !== 4) return
      if (xhr.status === 200 || xhr.status === 304) {
          resolve(xhr.responseText)
      } else {
          reject(new Error(xhr.responseText))
      }
    }
    xhr.send()
  })
}
```

# JSONP
script 标签不受同源策略约束, 所以可以执行跨域请求
优点是兼容性好, 缺点是只支持 GET 请求
```js
const jsonp = ({ url, params, callbackName }) => {
  const generateUrl = () => {
    let dataSrc = ''
    for (let key in params) {
      if (params.hasOwnProperty(key)) {
        dataSrc += `${key}=${params[key]}&`
      }
    }
    dataSrc += `callback=${callbackName}`
    return `${url}?${dataSrc}`
  }
  return new Promise((resolve, reject) => {
    const scriptEle = document.createElement('script')
    scriptEle.src = generateUrl()
    document.body.appendChild(scriptEle)
    window[callbackName] = data => {
      resolve(data)
      document.removeChild(scriptEle)
    }
  })
}
```
JSONP 需要服务端的配合, 这样的请求发出去后, 服务端应当返回这样的 result:
```js
"callbackName({\"userId\":1,\"username\":\"maos\"})"
```
script 标签在拿到这个东西后会调用全局 (window 上的) callbackName 函数
并把括号里的 JSON 数据作为这个函数的第一个参数 (需要 JSON.parse 一下)

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