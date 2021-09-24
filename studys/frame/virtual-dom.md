# 为什么使用虚拟 DOM
组件就是一个函数, 输入一些数据, 渲染对应的内容

Vue 一位作者说:
> 为何组件要从直接产出 html 变成产出 Virtual DOM 呢？其原因是 Virtual DOM 带来了 分层设计，它对渲染过程的抽象，使得框架可以渲染到 web(浏览器) 以外的平台，以及能够实现 SSR 等

> 至于 Virtual DOM 相比原生 DOM 操作的性能，这并非 Virtual DOM 的目标，确切地说，如果要比较二者的性能是要“控制变量”的，例如：页面的大小、数据变化量等

我觉得这个观点还是不太对, 最重要原因应该还是:
  1. 相比操作原生dom 更快 
  2. 相比模板引擎可以记录上一次状态实现diff更新
因为没有这种动力的话, 单凭跨平台的可能性是不足以推动它这么快这么大规模的流行的

# 自己实现虚拟 DOM
[Vue 核心开发者的文章](http://hcysun.me/vue-design/zh)

# 疑问
关于 双端比较 diff 算法比第一种 diff 算法的优势之处
理解的比较笼统, 还是没有想得太清楚, 需要写个实例来测试一下

# 更多
https://zhuanlan.zhihu.com/p/150732926
https://neil.fraser.name/writing/diff/
https://juejin.cn/post/6934698604955172877