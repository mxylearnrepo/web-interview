# Vue 服务端渲染
Vue 的 ssr 主要是针对 使用 vue 开发 spa 的 ssr.
主要解决了:
  - 提升 SPA 的首屏加载速度
  - 优化 seo
在前后端还没分离的时代, JAVA, PHP 这些老牌编程语言都是使用服务器渲染页面, 多年的沉淀已经发展出了成熟的 ssr 方案.
但是后来前后端分家了, 前端进入了 vue, react 时代, 而老牌语言额 ssr 只能在它们自己生态下做, 所以就需要新的 ssr 方案.

一个简单的案例:
```js
// server app.js
import Koa2 from 'koa'
import staticFiles from 'koa-static'
import { createRenderer } from 'vue-server-renderer'
import Vue from 'vue'
import App from './App.vue'
import { createRouter, routerReady } from './route'

const renderer = createRenderer()
const app = new Koa2()

// 静态资源
app.use(staticFiles('public'))

// 接管路由
app.use(async function (ctx) {
  // 创建 router
  const router = createRouter()
  router.push(ctx.req.url) // 把请求的 url 传入 router
  await routerReady(router) // 等待 router 钩子解析完毕

  // 创建一个 vue 实例 (组件)
  const vm = new Vue({
    // template: '<div>hello maos</div>'
    router,
    render: h => h(App) // router 中的 current 是响应式的, 故当 router.push 后 App 中渲染的 view 也会变化
  })

  // 获取 router 匹配上的 views
  // 注意: 这里匹配的 views 都是根部 page 级的组件
  const matchedComponents = router.getMatchedComponents()

  // 获取数据
  await Promise.all(
    matchedComponents.map(component => {
      if (component.asyncData) {
        return component.asyncData() // 这是一个 promise 实例
      }
    })
  )

  if (!matchedComponents.length) {
    ctx.body = '没找到网页, 404 ~'
    return
  }

  // 把这个组件转换成一个 html 字符串
  const context = {} // 创建一个上下文对象, 用于存放 render 产生的一些其他东西 (包括 styles)
  const html = await renderer.renderToString(vm, context) // 这时候 vm 内部的 view 已经响应式的变为新的

  ctx.set('Content-type', 'text/html;charset=utf-8')
  ctx.body = `
    <html>
      <head>
      ${context.styles ? context.styles : ''}
      </head>
      <body>
        ${html}
        <script src="/bundle.js"></script>
      </body> 
    </html>
  `
})

app.listen(8888) // 开始监听
```

> 集成 vue-route:
```js
// route.js
import Vue from 'vue'
import Router from 'vue-router'
import List from './pages/List' // 某个组件
import Search from './pages/Search' // 另一个组件

Vue.use(Router)
export const createRouter = () => {
  return new Router({
    mode: 'history',
    routes: [
      {
        path: '/',
        component: List,
      }
      {
        path: '/list',
        component: List,
      },
      {
        path: '/search',
        component: Search,
      }
    ]
  })
}

export const routerReady = (router) => {
  return new Promise(resolve => {
    router.onReady(resolve)
  })
}
```

> 同构: 
为什么需要同构?
Vue 的代码在服务端执行了也 输出成 html 直接返回给客户端了, 为什么在客户端还要再执行一遍?
因为服务器渲染有它的局限性: 例如在 template 上绑定了一个事件, 这段代码必须在浏览器环境中也能执行;
服务器返回了一个填满了内容的 html 给客户端, 但是后面的事它就管不了.
为了实现同构 需要在服务端 app.js 之外 增加一个客户端脚本入口: 
```js
// index.js
import Vue from 'vue'
import App from '../App.vue'
import { createRouter } from './route'

const router = createRouter()

new Vue({
  router,
  render: h => h(App)
}).$mount('#app', true) // 参数 true 是为了让这个 vue 实例只对当前已 由服务端渲染好的 模板内容只添加一些事件绑定和功能支持等
```
利用 webpack 从这个入口进入打包出一份客户端 bundle.js 代码, 用 script 标签引入写在服务器 app.js 的 {{html}} 后面

> 请求数据
数据(接口)放在一个远程服务器上,
服务端可以在组件中添加一个 asyncData 方法:
```vue
// List.vue
<template>
...
</template>
<script>
export default {
  asyncData () {
    return new Promise(resolve => {
      // 获取异步 data 并将其赋值给组件 data 使其响应式的造成 vm 实例的重新渲染
      resolve()
    })
  }
}
</script>
```
客户端是用不到这个 asyncData 方法的, 自己该怎么请求就怎么请求.

> 集成 vuex:
vuex 可以很好地帮我们管理数据集 (非必须)
``` js
// store.js
import Vue from 'vue'
import Vuex from 'vuex'

Vue.use(Vuex)

export function createStore() {
  return new Vuex.store({
    state: {
      list: [],
      name: 'maos'
    },
    actions: {
      async getList ({ commit }, params) {
        let listData = await fetch('api/list', params)
        commit('setList', listData)
      }
    },
    mutations: {
      setList(state, data) {
        state.list = data || []
      }
    }
  })
}
```
现在客户端和服务端都引入了 vuex, 服务端的 store 里放着 List 组件需要的远程数据, 客户端的 store 里却是空的
这样一旦客户端 vue 开始了执行, 各种初始化操作就会 冲掉 服务端 store 里渲染出的 html 中的动态数据
这时候就需要有 脱水/注水:
```js
// 脱水: 服务端返回的 html 中
ctx.body = `
  ...
  <script>
    var context = {
      state: ${JSON.stringify(store.state)}
    }
  </script>
  ...
`
// 注水: 客户端 js 中
if (window.context && window.context.state) {
  store.replaceState(window.context.state)
}
```

> 设置 meta:
可以直接写死, 但要想让不同的 view 设置不同的 meta, 
也可以使用 vue-meta 更灵活更优雅地分别在服务端和客户端设置 meta, 就像:
```js
// 在一个 vue 组件内:
export default {
  metaInfo: {
    title: "列表页",
    meta: [
      { charset: "utf-8" },
      { name: "viewport", content: "width=device-width, initial-scale=1" },
    ],
  }
  ...
}
```
[参考](https://segmentfault.com/a/1190000012849210)


> vue-server-renderer 原理:
首先它里面是一个工场函数, 可用来得到一个 Renderer 实例
Renderer 实例可以用来完成一个 从 Vue 实例 -> HTML 字符串的转化

我们知道一个 Vue 实例的 template 会被编译 (运行时 or 构建时) 为 render 函数,
而 render 函数里面会使用 snabbdom 的 h 函数来递归制造出 vdom, 最后替换为 真实 dom

我们的 renderer 得到了这个 Vue 实例, 也就得到了它的 render 函数, 也就得到了它的 vnode 们;
但是我们不替换为真实 dom, 而是递归访问 vnode, 拼接到一个字符串中, 把它作为 html 返回给客户端.