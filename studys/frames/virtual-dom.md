# 为什么使用虚拟 DOM
简单来说, 虚拟 DOM 就是用 Js 数据结构来表示 DOM 结构, 它并没有真实挂载到 DOM 上

Vue 一位作者说:
> 为何组件要从直接产出 html 变成产出 Virtual DOM 呢？其原因是 Virtual DOM 带来了 分层设计，它对渲染过程的抽象，使得框架可以渲染到 web(浏览器) 以外的平台，以及能够实现 SSR 等

> 至于 Virtual DOM 相比原生 DOM 操作的性能，这并非 Virtual DOM 的目标，确切地说，如果要比较二者的性能是要“控制变量”的，例如：页面的大小、数据变化量等

我觉得这个观点还是不太对, 最重要原因应该还是:
> 操作数据结构 远比 通过和浏览器交互去操作 DOM 快, 但是虚拟 DOM 最终也是需要挂载到浏览器上变成真实 DOM 节点的, 因此虚拟 DOM 并没有使**必要的**操作 DOM 次数减少, 但是能够精确地获取**最小的, 最必要的**操作 DOM 行为的集合


# 自己实现虚拟 DOM
[Vue 核心开发者的文章](http://hcysun.me/vue-design/zh)

# 疑问
关于 双端比较 diff 算法比第一种 diff 算法的优势之处
理解的比较笼统, 还是没有想得太清楚, 需要写个实例来测试一下

# 更多
https://zhuanlan.zhihu.com/p/150732926
https://neil.fraser.name/writing/diff/
https://juejin.cn/post/6934698604955172877