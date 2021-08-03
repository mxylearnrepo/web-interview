# 单线程
Node.js 的单个实例在单个进程的 单个线程 中运行

## child_process
衍生子进程, 生出来的这个进程与 Node 实例运行的进程是独立的
可以生出异步进程, 也可以生同步进程

exec
execFile
fork
spawn
前几个方法都是对 spawn 的封装

## cluster
cluster 是对 child_process 的进一步封装, 它解决了一个更具体的问题:
为了利用多核系统, 建立一个 进程集群 来执行相同的任务 来处理负载

创建的这些进程共享一个服务器端口

创建的这些进程可以作为 Worker 类的实例获取

## worker_threads
这是线程了, 具体没用过
