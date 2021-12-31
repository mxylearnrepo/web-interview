# 日志切割
[pm2-logrotate](https://github.com/keymetrics/pm2-logrotate)
https://segmentfault.com/a/1190000022587579
https://segmentfault.com/a/1190000020339900
https://segmentfault.com/a/1190000021351573

原理?
通过 pm2.connect 召唤 (连接到) 守护进程, 然后搞出一个新的进程按一定频率进行日志检查与切割

配置?
先安装:
```shell
pm2 install pm2-logrotate
```
安装完模块后，您必须键入: 
```shell
pm2 set pm2-logrotate:<param> <value>
```
```m
max_size (Defaults to 10M): When a file size becomes higher than this value it will rotate it (its possible that the worker check the file after it actually pass the limit) . You can specify the unit at then end: 10G, 10M, 10K
  > 配置项默认是 10MB，并不意味着切割出来的日志文件大小一定就是 10MB，而是检查时发现日志文件大小达到 max_size，则触发日志切割。
retain (Defaults to 30 file logs): This number is the number of rotated logs that are keep at any one time, it means that if you have retain = 7 you will have at most 7 rotated logs and your current one.
  > 这个数字是在任何一个时间保留已分割的日志的数量，这意味着如果您保留7个，那么您将最多有7个已分割日志和您当前的一个
compress (Defaults to false): Enable compression via gzip for all rotated logs
  > 对所有已分割的日志启用 gzip 压缩
dateFormat (Defaults to YYYY-MM-DD_HH-mm-ss) : Format of the data used the name the file of log
  > 文件名格式化的规则
rotateModule (Defaults to true) : Rotate the log of pm2's module like other apps
  > 像其他应用程序一样分割 pm2模块的日志
workerInterval (Defaults to 30 in secs) : You can control at which interval the worker is checking the log's size (minimum is 1)
  > 您可以控制工作线程检查日志大小的间隔(最小值为1）单位为秒（控制模块检查log日志大小的循环时间，默认30s检查一次）
rotateInterval (Defaults to 0 0 * * * everyday at midnight): This cron is used to a force rotate when executed. We are using node-schedule to schedule cron, so all valid cron for node-schedule is valid cron for this option. Cron style :
  > 多久备份一次
*    *    *    *    *    *
┬    ┬    ┬    ┬    ┬    ┬
│    │    │    │    │    |
│    │    │    │    │    └ day of week (0 - 7) (0 or 7 is Sun)
│    │    │    │    └───── month (1 - 12)
│    │    │    └────────── day of month (1 - 31)
│    │    └─────────────── hour (0 - 23)
│    └──────────────────── minute (0 - 59)
└───────────────────────── second (0 - 59, OPTIONAL)
TZ (Defaults to system time): This is the standard tz database timezone used to offset the log file saved. For instance, a value of Etc/GMT+1, with an hourly log, will save a file at hour 14 GMT with hour 13(GMT+1) in the log name.
  > 时区（默认为系统时区）
```

问题?
是否只需安装一次, 然后每次 pm2 启动都会执行? 那如何分应用设置?
  - 首先 pm2 本身能不能托管多个应用?
    - 显然是可以的
  - 那么 pm2-logrotate 的设置也是全局的咯?
    - 恐怕是这样的
如何做到的不影响业务进程? 如果业务把 cpu 核数占满了怎么办?
  - 不同应用进程存在竞争
  - 不同进程应该是分时间段的占用资源
按天和按大小的策略, 怎么样能方便于后续检索? 保留多少天的日志?
  - 首先每天晚上零点强制分割一次, 然后是一旦超出了大小随时分割
  - 这样就会出现每一天不一定有多少个文件的情况, 所以无法设置保留多少天
  - 咋办呢? 只能是每日上限设置的比较大 (1G) 配合每天一次强制切割
把切割检查频率设置到多少合适?
  - 默认的可能比较频繁 (默认 30s 一次)
当前文件已经远超一个文件的大小限制怎么办?
  - 试一下看它是不是每次切一个限制大小的文件出来
执行切割的时候如果有背压怎么办?
  - 如果上一个问题是这样的, 那么它应该是不停滴执行切割, 直背压消失
  - 还是需要设置一个相对较大的单文件限制, 减少背压的可能性
是否使用多线程协同?
  - 
切割是否会造成日志丢失?
  - 
日志文件复制时是否会导致磁盘占用量暴增? 如何尽可能减少?
  - 
要不要把错误日志和普通日志分离开?
  - 我觉得可以不分离, 便于查看错误日志产生的上下文
