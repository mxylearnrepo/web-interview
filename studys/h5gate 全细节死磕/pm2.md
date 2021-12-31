# github
https://github.com/Unitech/pm2

# 为什么用?
pm2 可以不侵入我们代码地, 为我们的应用提供多核和监控能力, 节省了很多精力

# 为什么不用?
如果我们的业务有特殊性, 就不能使用 pm2 了
最典型的就是**有状态服务**
很多带有 session 的 Node 应用, 在 pm2 负载均衡下, 可能丢失 session
还有一些 socket 应用, 需要加入同一个房间, 在同一个房间内共享数据, 也不能用 pm2
这时候就需要我们自己使用[cluster](../../web-interview/docs/NodeJS/cluster.u.md) 来手动处理了

# 源码结构
lib
├── API     # 日志管理、GUI 等辅助功能
├── God     # 多进程管理逻辑实现位置
└── Sysinfo # 系统信息采集

几个比较关键文件:
  - Daemon.js
    - 守护进程逻辑实现
  - God.js
    - 业务进程的包裹层
  - Client.js
    - 我们执行 pm2 命令的进程逻辑
  - API.js
    - 各种各样的具体功能逻辑
  - binaries/CLI.js
    - 执行 pm2 命令的时候的入口

# pm2 start app.js
我们把 pm2 安装到全局, 然后用 pm2 替代 node 启动我们的应用
这个过程中发生了什么呢?

首先我们这个命令 是执行的 lib/binaries/CLI.js
里面一个 pm2 实例被初始化了, 上面挂满了各种属性, 包括一个 Client 的实例
然后调用 pm2 的 connect 方法去 ping Daemon 进程, 如果没有 Daemon 进程, 就创建一个, 并与之建立 rpc 链接
在 pm2.connect 的回调函数里面, 使用 commander 处理输入的指令 (start app.js)
在这里就可以使用 rpc 的方式和 Daemon 进程交互, 实现任何愿望 (比如启动业务进程)

# pm2.connect
```js
pm2.connect(function() {
  // ...
})
```
这个方法是在干啥呢?
从找到的资料看, 这个方法内部调用了 client.start 方法, 大概是:
```js
this.pingDaemon(function (daemonAlive) { // 尝试连接 Daemon 进程
  // ...
  that.launchDaemon(function (err, child) { // 创建一个 Daemon 进程
    // ... 通过 child_process.spawn 
    that.launchRPC(function (err, meta) { // 与 Daemon 进程建立通信
      // ... 执行回调? 
    })
  })
})
```
目测就是执行一个召唤 Daemon 的仪式, 建立起一个与守护进程的联系

# 多进程管理
对 [Cluster 模块](../../web-interview/docs/NodeJS/cluster.u.md) 的封装
设置了 N 多的默认参数, 添加了亿点细节, 如在进程启动成功, 出现异常等等情况是的对应处理

比如进程重启:
守护进程 Daemon 维护管理与所有其他进程的连接
子进程可以监听到异常事件, 向守护进程发送消息终止进程
同时守护进程在接收到消息后, 会重新创建新的进程, 实现自动重启

# 系统信息监控
pm2 监控使用了 pidusage 这么个库
至于使用 pm2 monit, pm2 ls --watch 命令时, 就是定时器在循环调用上述 pidusage 的能力

# 日志管理
pm2 的日志包括 业务进程的日志 和 守护进程的日志

守护进程的日志 hack 了 console.log 增加了一个 发布订阅模型的消息传递

业务进程的日志则是通过 覆盖 process.stdout, process.stderr 对象上的方法
在接收到日志后写入一个文件里 (Stream), 同时调用 process.send 将日志进行转发, 从而使 Client 进程可以展示日志

[日志切割工具 pm2-logrotate](./pm2-logrotate.md)

# 总结
Client 进程 <-> 守护 (Daemon) 进程 <-> 业务进程

# 参考
https://segmentfault.com/a/1190000021230376
https://tn710617.github.io/pm2/
https://juejin.cn/post/6959235603523190821
https://juejin.cn/post/6844903545636929544
https://juejin.cn/post/6866081343454773262
