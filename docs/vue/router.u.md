# 说一下 router?
## 基本使用
```jsx
// router.js 导出 VueRouter 的一个实例
import Vue from 'vue'
import VueRouter from 'vue-router'
import Index from '../views/Index.vue'

Vue.use(VueRouter) // Vue 先执行这个 VueRouter 函数或者它里面的 install 函数

const routes = [
  {
    path: '/',
    name: 'Index',
    component: Index
  }
] // 路由规则映射
const router = new VueRouter({
  routes
})

export default router

// app.js 中注册这个 router 对象
import Vue from 'vue'
import App from './App.vue'
import router from './router'

new Vue({
  router, // 会给 vm 实例注入 $route (存储路由规则数据) 和 $router (路由 api 对象)
  render: h => h(App)
}).$mount('#app')

// App.vue 路由导航
<template>
  <div id="app">
    <div id="nav">
      <router-link to="/">Index</router-link>
      <router-link to="/blog">Blog</router-link>
      <router-link to="/about">About</router-link>
    </div>
    <!-- 通过 router-view 设置路由 view 占位 -->
    <rotuer-view />
  </div>
</template>
```
* 动态路由
```js
const routes = [
  {
    path: '/detail/:id', // 动态占位, id 是一个参数
    name: 'Detail',
    // 开启 props 会把 url 中的"参数"(id) 通过 props 传递给组件
    // 就不需要从 $route.params.id 中取了, 这样更合理, 因为它本来就是伪参数
    // 并且不会使我们的组件 (view) 强依赖与路由的这个"参数", 否则它只能使用在一个配置了路由的地方
    props: true,
    // 详情页这种 view, 大部分是不会被查看的
    // 所以没有必要在运行时初始化路由的时候就要获取到这个 view 的代码
    component: () => import(/* webpackChunkName: "detail" */ '../views/Detail.vue')
  }
]
```
* 嵌套路由
解决多个路由组件有相同的内容, 看起来像一个 slot.
例如在一个 Layout view 组件中:
```js
<template>
  <div class="head">Header</div>
  <div>
    <router-view></router-view>
  </div>
  <div class="foot">Footer</div>
</template>
```
配置:
```js
const routes = [
  {
    path: '/',
    component: Layout,
    children: [
      {
        name: 'Index',
        path: '', // path 为空的时候, router-view 会默认先加载它
        component: Index
      },
      {
        name: 'Detail',
        path: '/detail/:id', // 当 path 变成这样, 就会加载 detail view
        component: () => import('@/views/Detail.vue')
      }
    ]
  }
]
```

* 编程式导航
```js
  vm.$router.push({ name: 'Index' })
  vm.$router.replace({ name: 'Index' })
  vm.$router.go(-1)
```

* hash 模式 & history 模式
hash 模式中带 # 号, 基于锚点以及 onhashchange 事件;
history 模式是一个正常的 url, 基于 History API:
```js
history.pushState() // 改变浏览器地址栏中的 url, 放入回退栈, 但不会发送请求
history.replaceState()
```
history 模式需要服务器的支持, 因为我们输入浏览器地址栏的 **url 在 服务端不存在**.
这个时候如果只是 url 变了, 那么没有问题, 但变了之后如果**再刷新浏览器**, 浏览器会向服务端发请求.
所以在服务端应该配置: **除了静态资源之外的路径都返回 index.html**.
此外, 还要配置一个 404 路由:
```js
const routes = [
  {
    path: '*',
    name: '404',
    component: () => import(/* webpackChunkName: "404" */ '../views/404.vue')
  }
]

const router = new VueRouter({
  mode: 'history', // 因为默认是 hash 模式
  routes
})
```

服务端支持方式:
- 在 Node.js 中配置
```js
const path = require('path')
const history = require('connect-history-api-fallback') // vue 处理 history 模式的模块
const express = require('express') 

const app = express()
// 注册 history 中间件
app.use(history())
// 注册静态资源处理中间件
app.use(express.static(path.join(__dirname, '我们的 html 静态资源目录')))

app.listen(3000, () => {
  console.log('服务器开启, 端口: 3000')
})
```
- 在 Nginx 中配置
下载 Nginx 压缩包, 解压它到服务器目标根目录;
切到 Nginx 的安装目录, 启动它 (默认 80 端口):
```shell
# 启动
start nginx

# 重启
nginx -s reload

# 停止
nginx -s stop
```
每次修改 Nginx 配置文件, 需要重启它.
配置 history 模式:
```nginx
...
server {
  ...
  location / {
    root html;
    index index.html index.htm;
    try_files $uri $uri/ /index.html;
  }
  ...
}
...
```
<br>


## 实现原理
首先通过 history.pushState 往浏览器地址栏放入一个新 url, 这时不会触发刷新;
监听 popState 事件, 根据当前地址栏路由地址找到对应 view 组件渲染到 router-view 里.

popState 事件触发条件是什么?
浏览器前进后退, 有的浏览器在页面加载时也会触发.

pushState 与 replaceState 的区别是什么?
前者是往后加, 后者替换当前的一条.

```js
let _Vue = null
export default class VueRouter {
  constructor (options) {
    this.options = options
    this.routeMap = {} // key: 路由地址, val: 对应组件
    this.data = _Vue.observable({ // 帮我们创建一个响应式的对象, 路由地址发生改变, 对应的 view 组件要发生更新
      current: '/' // 默认是当前地址: /
    })
  }

  static install (Vue) { // Vue: Vue 的构造函数
    // 1. 判断当前插件是否已经被安装
    if (VueRouter.install.installed) {
      return 
    }
    VueRouter.install.installed = true

    // 2. 把 Vue 构造函数注册到全局变量 (实际上是模块的函数作用域)
    _Vue = Vue

    // 3. 把第一个创建的 Vue 实例时传入的 router 对象注入到所有 Vue 实例上
    // _Vue.prototype.$router = ?
    _Vue.mixin({ // 接下来所有的 Vue 实例都被混入了一个 beforeCreate 钩子
      beforeCreate () {
        // 但是并不是每个 Vue 实例都要执行这个钩子, 它只应该执行一次, 所以加个判断:
        if (this.$options.router) { // 只有第一个 Vue 实例有这个 router 属性
          _Vue.prototype.$router = this.$options.router
          this.$options.router.init() // 初始化这个全局唯一 router 实例
        }
      }
    })
  }

  init () {
    this.createRouteMap()
    this.initComponents(_Vue)
    this.initEvent()
  }

  initEvent () { // 注册 popstate 事件
    // 当我们点击浏览器前进后退的时候, 需要监听并且处理
    window.addEventListener('popstate', () => {
      this.data.current = window.location.pathname
    })
  }

  createRouteMap () { // 初始化 routeMap 属性
    // 遍历 options.routes 数组, 解析成键值对的形式, 存储到 routeMap 对象里
    this.options.routes.forEach(route => {
      this.routeMap[route.path] = route.component
    })
  }

  initComponents(Vue) { // 创建 router-link 与 router-view 组件
    // router-link
    Vue.component('router-link', {
      props: {
        to: String
      },
      // 运行时版本不带 templateCompiler 的 vue 不支持 template 选项
      // template: '<a :href="to"><slot></slot></a>' 
      methods: {
        clickHandler (e) {
          history.pushState({}, '', this.to)
          this.$router.data.current = this.to
          e.preventDefault()
        }
      },
      render (h) { // 运行时直接执行 render 函数
        return h('a', { // this 在 render 函数的上下文中
          attrs: {
            href: this.to
          },
          on: {
            click: this.clickHandler
          }
        }, [this.$slots.default]) // 神奇
      }
    })

    // router-view
    const that = this
    Vue.component('router-view', {
      render (h) { // 当 routeMap 对象中的 current 发生改变, vue 会重新执行这个 render
        // 1. 找到当前路由的地址
        const current = that.data.current
        // 3. 再根据地址从 routeMap 中找到对应的 view 组件
        const view = that.routeMap[current]
        // 3. 再把这个 view 组件转为虚拟 dom 直接返回
        return h(view)
      }
    })
  }
}
```

## hash 模式实现
```js
let _Vue = null
export default class VueRouter {
  constructor (options) {
    this.options = options
    this.routeMap = {} // key: 路由地址, val: 对应组件
    this.data = _Vue.observable({ current: '' }) // 当前 hash 为 ""
  }

  static install (Vue) { // Vue: Vue 的构造函数
    // 1. 判断当前插件是否已经被安装
    if (VueRouter.install.installed) return
    VueRouter.install.installed = true

    // 2. 把 Vue 构造函数注册到全局变量中 (router 模块的函数作用域下)
    _Vue = Vue

    // 3. 把第一个创建的 Vue 实例时传入的 router 对象注入到所有 Vue 实例上
    _Vue.mixin({ // 接下来所有的 Vue 实例都被混入了一个 beforeCreate 钩子
      beforeCreate () {
        // 但是并不是每个 Vue 实例都要执行这个钩子, 它只应该执行一次, 所以加个判断:
        if (this.$options.router) { // 只有第一个 Vue 实例有这个 router 属性
          _Vue.prototype.$router = this.$options.router
          this.$options.router.init() // 初始化这个全局唯一 router 实例
        }
      }
    })
  }

  init () {
    this.createRouteMap()
    this.initComponents()
    this.initEvent()
  }

  createRouteMap () { // options.routes 数组 -> routeMap 键值对
    this.options.routes.forEach(route => {
      this.routeMap[route.path] = route.component
    })
  }

  initEvent () {
    window.addEventListener('hashchange', function () {
      this.data.current = window.location.hash
    })
    window.addEventListener('click', function(e) {
      if (e.target.nodeName === 'a') { // 处理一下 a 标签
        location.href = '#' + e.target.href
        // this.$router.data.current = e.target.href
        e.preventDefault()
      }
    })
  }

  initComponents () { // router-link &  router-view
    Vue.component('router-link', {
      props: {
        to: String
      },
      methods: {
        clickHandler (e) {
          location.href = '#' + this.to
          // this.$router.data.current = this.to
          e.preventDefault()
        }
      },
      render (h) {
        return h('a', {
          attrs: { href: this.to },
          on: { click: this.clickHandler }
        }, [this.$slots.default])
      }
    })

    const that = this
    Vue.component('router-view', {
      render (h) {
        // 1. 找到当前路由的地址
        const current = that.data.current
        // 3. 再根据地址从 routeMap 中找到对应的 view 组件
        const view = that.routeMap[current]
        // 3. 再把这个 view 组件转为虚拟 dom 直接返回
        return h(view)
      }
    })
  }
}
```
