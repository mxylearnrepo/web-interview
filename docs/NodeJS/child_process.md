# child_process
Node 最基础的创建进程库, 包括
  spawn
  exec
  execFile
  fork
四个方法

## spawn
默认不会创建 shell 去执行传入的命令, 所以性能上稍好一点
基于 stream 实现 IPC 等

## exec
会创建一个 shell, 完全支持 shell 语法, 可传入任何 shell 脚本
不基于 stream, 占用内存
存在 命令注入 的安全风险

## execFile
不通过 shell 执行, 要求传入可执行文件
同样不基于 stream, 会占用内存

## fork
spawn 的变体 (封装), 特点是父子进场自带通信机制 (IPC 管道)
所以尤其适合用来分析耗时逻辑

## xxxSync
所有 API 的同步阻塞版本

# IPC
进程间通信
通过stdin/stdout传递json

或者使用原生 IPC
Node 父进程
  - process.on('message') 收
  - child.send() 发
子进程
  - process.on('message) 收
  - process.send() 发

或者借助网络 sockets
  不仅能跨进程, 还能跨机器

或者 message queue
  相当于一个中间层

或者直接用 Redis
  与 message queue 类似