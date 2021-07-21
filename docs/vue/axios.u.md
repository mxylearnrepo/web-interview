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
  比如页面上有两个 tab, 切换每一个 tab 的时候会发起一次请求
  但是每个 tab 其实公用了一个数组来存放数据, 每次请求返回后清空该数组并装入新数据
  然后我点了一个 tab, 请求发出去了, 这时候异步获取数据的 ajax 需要一段时间来将数据传回并渲染到页面上
  这时候我手快又点了另一个 tab, 然而前一个请求的数据还没回来, 我们清空了一个 空数组
  这时候前一个请求的数据回来了, 后一个请求的数据随后也回来了, 我们往数组里 push 了两次俩 tab 的数据
  当然也可以在每次数据回来而不是请求发出的时候清空这数组, 但是如果还有上拉加载的话呢?
  所以就有了取消请求这个思路

取消请求是怎么做到的?
  浏览器端有原生的 XMLHTTPRequest.abort() 方法
  Node 平台的 http 模块应该也有相似的办法 (吧)


# axios 的 XSRF 防御
在 axios 中, 主要依赖双重 cookie 抵御 CSRF(XSRF)
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
https://www.axios-http.cn/docs/intro
https://lxchuan12.gitee.io/axios/
