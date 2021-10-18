[Vue nextTick实现原理](https://www.cnblogs.com/leiting/p/13174545.html)

# 一个误区
上面这篇介绍 Vue nextTick 原理的文章有一个误区
似乎使用 微任务 来实现 nextTick 的理由是 微任务 会在执行栈空闲的时候立即执行, 它的响应速度相比setTimeout会更快, 因为无需等渲染

其实是 Vue 为了实现 异步更新队列, 所以基于上面这个原因, 使用了 Promise.then, MutationObserver, 甚至可以降级为 setImmediate, setTimeout, 来实现这个队列
[参考](https://cn.vuejs.org/v2/guide/reactivity.html#%E5%BC%82%E6%AD%A5%E6%9B%B4%E6%96%B0%E9%98%9F%E5%88%97)

既然异步更新队列是这样实现的, 那么 nextTick 方法必然也会这么实现, 因为它要在队列执行完后执行回调