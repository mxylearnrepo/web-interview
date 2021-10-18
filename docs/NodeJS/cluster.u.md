# Cluster 也是多进程
Cluster 与 child_process 似乎重复做了一件事, 创建子进程
其实 Cluster 是对 child_process 的封装, 目的是实现一种负载均衡的任务调度

Cluster 服务于特定用途, child_process 则更加基础

Node.js 在官方示例中用 worker 代表 fork 出的子进程, 造成了很多人和浏览器中的 worker 多线程的混淆
而在 Node.js 中实现多线程的是 worker_threads

[cluster 源码解析](https://www.cnblogs.com/dashnowords/p/10958457.html)

# CPU
计算机由 CPU 来执行计算任务
如果一个计算机有多个 CPU, 多个进程可以并行在多个 CPU 中计算, 当然切换进程也存在开销
如果一个计算机只有一个 CPU, 那么进程在这个 CPU 中是并发运行的, 进程根据时间片轮换执行
但如果这个 CPU 有多个内核, 那么同一个进程下的多个线程可以并行执行, 如果内核多而线程少, 就出现浪费

# 进程与线程
进程就是程序, 每一个程序至少开辟一个新的进程, 这个进程可以派生子进程
线程可以看做是更细粒度的进程, 它享有所在进程的一切资源, 是进程的小克隆体
多线程需要依赖多内核 CPU 的支持
