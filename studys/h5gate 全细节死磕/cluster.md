# Cluster 也是多进程
Cluster 与 child_process 似乎重复做了一件事, 创建孩子进程

其实 Cluster 是对 child_process 的封装, 帮我们实现了子进程管理, 封装进程通信机制, 还有负载均衡等等功能, 这些功能要自己实现的话, 需要花费大量精力
所以 Cluster 服务于特定用途, child_process 则更加基础

Node.js 在官方示例中用 worker 代表 fork 出的子进程, 造成了很多人和浏览器中的 worker 多线程的混淆
而在 Node.js 中实现多线程的是 worker_threads

[cluster 源码解析](https://www.cnblogs.com/dashnowords/p/10958457.html)

# 使用
cluster.fork() 可以直接创建一个新的孩子进程
注意 Cluster fork 的数量最好不要超过 CPU 的核数
```js
let cluster = require('cluster')
let http = require('http')
let numCpus = require('os').cpus().length

if (cluster.isMaster) { // 主进程
  for (let i = 0; i < numCpus; ++i) {
    cluster.fork() // 创建 cpu 相同核数量的 子进程
  }
} else { // 子进程
  // 每个子进程创建一个 web 服务器实例
  // 每当有请求进来的时候, Cluster 会使用负载均衡算法将请求交给各个子进程
  http.createServer((req, res) => {
    res.writeHead(200)
    res.end('process ' + process.pid + ' says hello')
  }).listen(8888)
}
```

# 用途
除了利用计算机多核提升性能外, 还可以做到**零崩溃时间**
Cluster 可以检测到某个 worker 的崩溃 (这个 worker 不是指线程, 而是工作进程)
```js
cluster.on('exit', function(worker, code, signal) {
  console.log('Worker ' + worker.process.pid + ' died with code: ' + code + ' and signal: ' + signal)
  cluster.fork()
})
```

# CPU
计算机由 CPU 来执行计算任务
如果一个计算机有多个 CPU, 多个进程可以并行在多个 CPU 中并行, 当然切换进程也存在开销
如果一个计算机只有一个 CPU, 那么进程在这个 CPU 中是并发运行的, 进程根据时间片轮换执行
但如果这个 CPU 有多个内核, 那么多个线程可以并行执行, 如果内核多而线程少, 就出现浪费
如果一个进程只包括一个或不满多核数的线程, 那么多个进程也可以在多核 CPU 下并行工作

# 进程与线程
进程就是程序, 每一个程序至少开辟一个新的进程, 这个进程可以派生子进程
每一个进程用自己完全独立的数据存储空间, 堆栈空间等
线程可以看做是更细粒度的进程, 它享有所在进程的一切资源, 是进程的小克隆体
多进程多线程并行执行需要依赖多内核 CPU 的支持
