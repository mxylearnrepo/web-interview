# Koa 的 listen
Koa 的 listen 封装了 http 模块 createServer(cb) 返回的 net.Server 实例的 listen
这个 listen 接收一个 port 号和一个 host, host 在 IPv6 没配置的时候一般就是 0.0.0.0
0.0.0.0 代表了本机 (服务器) 所有可用的 IPv4 地址

# koa 中间件设计
先来看一下 Koa 中间件的使用方式
```js
const Koa = require('koa')
const app = new Koa()

// 一个中间件就是一个异步函数
// 最外层中间件, 可以用于兜底 Koa 全局错误
app.use(async (ctx, next) => {
  try {
    await next()
  } catch(error) {
    console.log(`[koa error]: ${error.message}`)
  }
})
// 第二层中间件, 用户记录日志
app.use(async (ctx, next) => {
  console.log(JSON.stringify(ctx.req))
  await next()
  console.log(JSON.stringify(ctx.res))
})
```
Koa 通过 use 方法注册的中间件
```js
// in Koa ...
use (fn) {
  this.middleware.push(fn)
  return this // 方便外部链式调用
}
```
那么中间件是怎么被执行的呢? 
我们 new Koa() 以后, 它首先会要创建一个 http 服务
```js
// in Koa ...
// 利用 http 模块的 API 创建一个 NodeJs 服务
listen(...args) {
  const server = http.createServer(this.callback())
  return server.listen(..args)
},
callback() {
  const fn = compose(this.middleware) // 把中间件们串起来
  const handleRequest = (req, res) => {
    const ctx = this.createContext(req, res) // 创建一个 context
    return this.handleRequest(ctx, fn)
  }
  return handleRequest
},
handleRequest(ctx, fnMiddlewares) {
  const res = ctx.res
  res.statusCode = 404

  const onerror = err => ctx.onerror(err)
  const handleResponse = () => respond(ctx) // 这个是真正执行响应的方法
  // 'on-finished' npm 包提供的方法
  // 该方法在一个 HTTP 请求 closes, finishes, or errors 时执行回调
  onFinished(res, onerror)
  return fnMiddlwares(ctx).then(handleResponse).catch(onerror)
}
```
上面有一个 compose 方法, 是 next 执行的关键
```js
function compose(middleware) {
  return function (ctx, next) {
    let index = -1

    function dispatch(i) { // dispatch 实际上是一个返回 Promise 的异步方法
      if (i <= index) return Promise.reject(new Error('next() called multiple times'))
      index = i // 这个 index 以及 middleware 都会被闭包缓存起来
      let fn = middleware[i] // 取出第 i 个中间件为 fn
      if (i === middleware.length) fn = next // 这个 next 是个啥??
      if (!fn) return Promise.resolve() // 取不到说明到结尾了

      try {
        // 执行下一个中间件方法
        // 这里可以看到, 我们在中间件里执行的 next 方法, 其实就是 dispatch.bind(null, i + 1)
        return Promise.resolve(fn(ctx, dispatch.bind(null, i + 1)))
      } catch (err) {
        return Promise.reject(err)
      }
    }

    return dispatch(0) // 从第一个中间件开始执行
  }
}
```
dispatch(n) 对应第 n + 1 个中间件的执行, 在中间件方法内部可以通过 await next() 来执行下一个中间件
在最后一个中间件执行完成后, 会依次再回流到之前的中间件, 直到第一个中间件中的代码执行完毕

# 所谓洋葱模型
Koa 的中间件被形象的称为洋葱模型
> 所谓洋葱模型, 就是指每一个 Koa 中间件都是一层洋葱圈, 它既可以掌管请求进入, 也可以掌管响应返回.
> 换句话说: 外层的中间件可以影响内存的请求和响应阶段, 内层的中间件只能影响外层的响应阶段.

洋葱模型**并不是**指核心操作 (respond) 在洋葱芯的地方执行, 也就是说**并不是**在最后一个中间件执行中执行
从上面代码可以看出, Koa 是在所有中间件不停滴 then 完毕, 也就是回到第一个中间件并执行完毕后, 才执行 respond

洋葱模型的优点是, 对于日志记录以及错误处理等需求非常友好

Koa1 版本中的中间件实现, 没有使用 Promise (async/await), 而是利用了 Generator 函数 + Co 库
本质上 Koa1 版本和 Koa2 中间件思想是一致的, 而 Koa2 的实现更加简洁, 优雅

# 对比 express
Express 不同于 Koa, 它集成了 路由, 静态服务器, 模板引擎 等功能, 因此看上去比 Koa 更像一个框架
  
Express 同样通过 app.use 注册中间件
不同的是 Express 的中间件设计并不是一个洋葱模式, 它是基于回调实现的线性模型 (一个中间件回调下一个)
如果想实现一个记录请求响应的中间件, 并不像 Koa 一样简单
```js
var express = require('express')
var app = express()
var requestTime = function (req, res, next) {
  req.requestTime = Date.now()
  next()
}
app.use(requestTime)
app.get('/', function (req, res) {
  var responseText = 'Hello World!<br>'
  responseText += '<small>Requested at: ' + req.requestTime + '</small>'
  res.send(responseText)
})
app.listen(3000)
```
可以看到, 上述实现就对业务代码有一定程度的侵入, 甚至可能造成不同中间件的耦合
Koa 的洋葱模型无疑更加先进, 而 Express 的线性机制不容易实现拦截处理逻辑, 比如异常处理和统计响应时间等


# 实现一个洋葱化的 fetch 库
[axios 中实现的拦截器](./axios.md)是将 request 和 response 中间件分开实现的

不过 axios 并没有按照洋葱模型实现, 但是现在我们学 Koa 设计一个洋葱式的 fetch 库
Koa 相当于是只需要对 response 过程做中间件设计, 而 fetch 则需要像 axios 一样分别实现中间件

要设计一个东西, 首先要设计它的预期使用方式:
```js
import Onion from 'onion' // 我们的洋葱
const fetchLib = {
  interceptors: {
    req: new Onion(),
    res: new Onion(),
  }
  request: async params => {
    this.interceptors.req.execute(params) // 执行 request 中间件s
    const response = await fetch(params) // 可以是原生 fetch 或者 ajax 或者通过适配器去定义
    this.interceptors.res.execute(response) // 执行 response 中间件s
  }
}

// 一个处理 URL 的中间件
fetchLib.interceptors.res.use(async (params, next) => {
  params.url = params.url.replace(/^(http:)?/, 'https:')
  await next()
  console.log(params.url) // 可以方便的检查后面的中间件有没有把我们的 url 再改掉
})

// 一个做错误处理中间件
fetchLib.interceptors.res.use(async (response, next) => {
  try {
    await next()
  } catch(error) {
    console.error(`[fetch response error]: ${error.message}`)
  }
})

// 一个处理返回结果的中间件
fetchLib.interceptors.res.use(async (response, next) => {
  if (!response.ok) {
    throw new Error(response.status + ' ' + response.statusText);
  }
  await next()
  if (/application\/json/.test(response.headers.get('content-type'))) {
    return response.json()
  }
})

// 做请求
fetchLib.request({
  url: '...'
})

```
然后来实现它:
```js
function compose(middlewares) {
  return function (params) {
    let index = -1
    function dispatch(i) {
      index = i
      const fn = middlewares[i]
      if (!fn) return Promise.resolve()
      return Promise.resolve(fn(params, () => dispatch(i + 1)))
    }
    return dispatch(0)
  }
}

export default class Onion {
  constructor() {
    this.middlewares = []
  }

  // 添加中间件s
  use(newMiddleware) {
    this.middlewares.push(newMiddleware)
  }

  // 执行中间件
  execute(params = null) {
    const fn = compose(this.middlewares)
    return fn(params)
  }
}
```

那么 fetch 库为什么也应该用中间件实现呢?
fetch 库的核心是实现发送请求, 而各种业务逻辑都可以以中间件化的形式对库进行增强
这样就实现了特定业务需求和请求库的解耦, 和分层