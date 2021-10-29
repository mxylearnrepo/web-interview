# webpack 为什么慢?
有大佬分析过, 说是 webpack 打包流程的瓶颈在于 代码构建 和 代码压缩 两个阶段

1.代码构建
做的就是分析模块依赖, 生成一份 Module Graph
这件事非常复杂, 也很耗时
这部分的详细解析, [有个视频讲的不错](https://youtu.be/Lzh8A0p3z8g)

有人说这个过程的缓慢是由于 Js 语言的原因
我觉得和语言并没关系, 而是和语言的执行方式有关
C/C++ (浏览器) 以及 Go (esbuild) 之所以快是因为他们的代码可以直接执行 (编译后)
而在 esbuild 开始解析 Javascript 代码时, node 还在忙于解析打包程序自身的 Javascript
等 Webpack 开始进行依赖分析, esbuild 可能都打包结束了

另外, Go 在线程之间可以共享内存, 而 Js 在线程之间必须序列化数据, 这又是一层损耗

2.代码压缩
Webpack 压缩代码用的是 terser, 它先分析 js AST, 去掉无效代码, 去掉 console.logs
terser 的毛病就是速度非常慢, 也就拖慢了 Webpack 的速度
但是 terser 人家虽然慢, 但是成熟稳定, 处理出来的文件体积非常小 (比 esbuild 要小哦!)


# esbuild 为什么快?
esbuild 所有东西都是自己实现的, 没有使用第三方依赖
这样可以带来很多性能优势, 所有的模块设计都为统一的性能目标服务
也可以最少次数的访问 AST, 因为所有步骤都可以因为共享内存而并行化的交织在一起

但是并不代表 esbuild 不支持插件, 事实上是支持的, 但是显然不如 Webpack 和 rollup 灵活

# esbuild-loader/plugin 原理
[浅谈esbuild-loader的原理](https://juejin.cn/post/6844904153890684936)