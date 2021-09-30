# axios 值得借鉴的地方
## 拦截器设计
把所有功能都在一个 get 函数里统一处理, 会导致 get 函数臃肿, 不易扩展
于是 axios 设计了拦截器来拆分一个请求, 其实很像 koa 的洋葱模型:
  请求拦截器: 在请求发送前统一执行某些操作, 比如在 header 里添加 token
  响应拦截器: 在服务器响应后统一执行某些操作, 比如 401 时自动跳转登录页

拦截器的本质就是一个实现特定功能的函数 (好像所有东西的本质都是一个函数)
拦截器的执行过程是: 任务注册 -> 任务编排 -> 任务调动 三个阶段
拦截器是不是等同于中间件?

## 适配器设计
对于浏览器, 可以通过 XMLHTTPRequest 或 fetch api 来发送 http 请求
对于 Node, 可以通过 http 或 https 模块来发送 http 请求

适配器的本质也是一个函数, 它返回一个 Promise 实例 

而我们还可以自定义自己的适配器, 什么时候需要自定义适配器呢?
当我们不想通过 ajax or http 来做请求的时候, 比如 mock 假数据
就好像虚拟 DOM 的实现中可以自定义各种平台 APIs 来实现不同的功能似的

# axios 一小段源码分析
哇哦, 来
```js
// lib/core/Axios.js
function Axios(instanceConfig) {
  this.defaults = instanceConfig;
  this.interceptors = {
    request: new InterceptorManager(),
    response: new InterceptorManager()
  };
}
// lib/core/InterceptorManager.js
function InterceptorManager() {
  this.handlers = [];
}
InterceptorManager.prototype.use = function use(fulfilled, rejected) {
  this.handlers.push({
    fulfilled: fulfilled,
    rejected: rejected
  });
  // 返回当前的索引，用于移除已注册的拦截器
  return this.handlers.length - 1;
};
Axios.prototype.request = function request(config) {
  config = mergeConfig(this.defaults, config);
  // ...
  var chain = [dispatchRequest, undefined]; // req 与 res中间是 dispatchRequest
  var promise = Promise.resolve(config);
  // 任务编排
  this.interceptors.request.forEach(function (interceptor) {
    chain.unshift(interceptor.fulfilled, interceptor.rejected);
  });
  this.interceptors.response.forEach(function (interceptor) {
    chain.push(interceptor.fulfilled, interceptor.rejected);
  });
  // 任务调度
  while (chain.length) {
    promise = promise.then(chain.shift(), chain.shift());
  }
  return promise;
};

// lib/core/dispatchRequest.js
module.exports = function dispatchRequest(config) {
  // ...
  var adapter = config.adapter || defaults.adapter;

  return adapter(config).then(function (response) {
    // ...
    return response;
  }, function (reason) {
    // ...
    return Promise.reject(reason);
  });
};

// lib/defaults.js
module.exports = defaults
var defaults = {
  adapter: getDefaultAdapter(),
  // ...
};
// ...
function getDefaultAdapter() {
  var adapter;
  if (typeof XMLHttpRequest !== 'undefined') {
    // 浏览器端使用 XMLHttpRequest 方法
    adapter = require('./adapters/xhr'); // 浏览器平台自己使用 Ajax
  } else if (typeof process !== 'undefined' && 
    Object.prototype.toString.call(process) === '[object process]') {
    // Node.js 端，使用 HTTP 模块
    adapter = require('./adapters/http'); // nodejs 平台自己使用 http 模块
  }
  return adapter;
}
```
一个 adaper 就是一个返回一个 Promise 实例的函数, 大概长这样:
```js
module.exports = function myAdapter(config) {
  // ...
  return new Promise(function(resolve, reject) {
    // ...
    sendRequest(resolve, reject, response); // 平台 API
    // ....
  });
}
```

# axios 为什么需要取消
既然请求了一个接口, 为什么又要取消, 岂不是多此一举?
需要取消的场景:
  比如页面上有个按钮, 点击后发起一个 http 请求, 快速点击会重复发起请求
  比如页面上有不同 tab, 快速切换 tab 会发起重复请求, 扰乱当前数据状态

## 取消请求是怎么做到的?
浏览器端有原生的 XMLHTTPRequest.abort() 方法, Node 平台的 http 模块应该也有相似的办法 (吧)
```js
let xhr = new XMLHttpRequest()
xhr.open('GET', url, true)
xhr.send()
setTimeout(() => xhr.abourt(), 300)
``` 

而在 axios 中封装的使用方式如下:
```js
const CancelToken = axios.CancelToken
const source = CancelToken.source()

axios.post(url, params, {
  cancelToken: source.token
})

source.cancel('canceled by the user') // 取消请求
```

也可以通过调用 CancelToken 的构造函数来创建 token:
```js
const CancelToken = axios.CancelToken
let cancel = null

axios.get(url, {
  cancelToken: new CancelToken(function executor(c) {
    cancel = c
  })
})
cancel() // 取消请求
```

## 怎么判断重复请求呢?
一个请求, 它的 URL 地址和请求参数如果是一样的, 就可以认为是同一个请求
所以在发起请求时, 可以用请求的 url 和 params 生成一个唯一的 key, 再给它生成一个 cancelToken
把 key 和 token 存储到一个 map 中, Javascript 的 Map 内置类型可以快速判断是否有重复的
```js
import qs from 'qs'

const pendingRequest = new Map();
// GET -> params；POST -> data
const requestKey = [method, url, qs.stringify(params), qs.stringify(data)].join('&'); 
const cancelToken = new CancelToken(function executor(cancel) {
  if(!pendingRequest.has(requestKey)){
    pendingRequest.set(requestKey, cancel);
  }
})
```

## CancelToken 的原理
```js
// lib/cancel/CancelToken.js
function CancelToken(executor) {
  if (typeof executor !== 'function') {
    throw new TypeError('executor must be a function.');
  }

  var resolvePromise;
  this.promise = new Promise(function promiseExecutor(resolve) {
    resolvePromise = resolve;
  });

  var token = this;
  executor(function cancel(message) { // 设置cancel对象
    if (token.reason) {
      return; // Cancellation has already been requested
    }
    token.reason = new Cancel(message);
    resolvePromise(token.reason);
  });
}

// lib/adapters/xhr.js 
if (config.cancelToken) {
  config.cancelToken.promise.then(function onCanceled(cancel) {
    if (!request) { return; }
    request.abort(); // 取消请求
    reject(cancel);
    request = null;
  });
}
```

# axios 的 XSRF 防御
在 axios 中, 主要依赖双重 cookie 抵御 [CSRF(XSRF)](../JavaScript/security.u.md)
所谓的双重 cookie 就是服务端在请求域名中注入一个 cookie, 
前端在自己的请求头或 URL 参数中再加上这个 Cookie,
后端接口拿到俩 cookie (cookie 自带一个, 前端请求带一个) 来验证是否合法

看 axios 的实现
```js
// lib/defaults.js
var defaults = {
  adapter: getDefaultAdapter(),
  // ...
  xsrfCookieName: 'XSRF-TOKEN',
  xsrfHeaderName: 'X-XSRF-TOKEN',
};
```
axios 默认配置了默认 xsrfCookieName 和 xsrfHeaderName, 实际开发中可以按具体情况传入配置


# 更多
[Axios 源码分析](../../docs/vue/axiosCode.md)
https://www.axios-http.cn/docs/intro
https://lxchuan12.gitee.io/axios/
