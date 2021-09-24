# Router
根据 url 的变化去自动改变 view 状态, 就是把响应性扩展到了地址栏

# Store
[简单store模式](https://cn.vuejs.org/v2/guide/state-management.html#%E7%AE%80%E5%8D%95%E7%8A%B6%E6%80%81%E7%AE%A1%E7%90%86%E8%B5%B7%E6%AD%A5%E4%BD%BF%E7%94%A8)

# Mixin
mixin 的本质还是对象之间的合并，但是对不同对象和方法右不同的处理方式
对于普通对象, 就是简单的对象合并类似于Object.assign,对于基础类型就是后面的覆盖前面的
而对于生命周期上的方法, 相同的则是合并到一个数组中, 调用的时候依次调用

# 服务端渲染
主要解决了:
  - 提升 SPA 的首屏加载速度
  - 优化 SEO 对爬虫引擎友好
在前后端还没分离的时代, JAVA, PHP 这些老牌编程语言都是使用服务器渲染页面, 多年的沉淀已经发展出了成熟的 ssr 方案
但是后来前后端分家了, 前端进入了 vue, react 时代, 而老牌语言额 ssr 只能在它们自己生态下做, 所以就需要新的 基于框架 ssr 方案
而这些框架的 ssr 主要是针对使用框架实现 spa 应用的 ssr 方案, 如果是传统多页, 不一定要按照框架的思路去实现服务端渲染

# 一些疑问
* Vue 为什么不需要时间分片?
https://juejin.cn/post/6844904134945030151
https://zhuanlan.zhihu.com/p/133819602

* 虚拟 DOM 与 React Fiber
https://www.zhihu.com/question/31809713
https://www.zhihu.com/question/31809713
https://zhuanlan.zhihu.com/p/26027085
https://www.cnblogs.com/WindrunnerMax/p/14781432.html
https://www.jb51.net/article/209310.htm

* 其他
https://segmentfault.com/a/1190000019292569  
