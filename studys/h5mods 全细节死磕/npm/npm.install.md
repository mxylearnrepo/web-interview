# npm2 与 npm3
npm2 所有包的依赖是嵌套关系, 造成了"嵌套地狱", 嵌套过多, 过深
npm3 会尽量将所有包的依赖"平铺"到 node_modules 中, 

如果有的包依赖了不同版本的模块, 则看是不是 node_modules 下已经有这个模块
如果有就在依赖者自身的 node_modules 下面加上自己需要的版本的依赖模块包

而对于 [peerDependencies](https://xwenliang.cn/p/5af2a97d5a8a996548000003), 
npm2 是强制安装的 (即使没有在宿主环境的 package.json 中显式声明, 也会被安装在宿主的 node_modules 中)

而 npm3 不再强制安装而是给出警告
这是因为 npm3 "平铺" 之后, 需要手动根据警告调整第一层依赖的包, 否则就冲突了
咋调整呢, 在宿主环境的 package.json 中显式声明呗, 然后冲突的模块会被安装在依赖它的包下面去

所以 npm3+ 说起来只是对 不冲突 的情况下的包进行了扁平化, 如果冲突依赖, 还是会有很多嵌套

# npm install 原理
当我们输入 npm install 并按下回车键时

> 检查并获取 npm 配置文件 (.npmrc)

> 检查当前工作目录是否有 lock 文件
如果有, 则检查 lock 文件和 package.json 声明的依赖是否一致
  如果一致, 直接用 lock 中的信息从缓存或网络加载依赖包
  如果不一致, npm5以下根据 lock 加载, 以上根据 package.json 加载并更新 lock 文件
如果没有则根据 package.json 加载模块并生成 lock 文件

> 确定首层依赖模块
首先要找的是宿主 (当前 npm 模块) 的 package.json 中的 dependencies 和 devDependencies 中直接指定的模块

宿主本身是整个依赖树的根, 每个首层依赖就是根节点下的一个子树, 此时 npm 会开启多进程从每个首层依赖模块开始逐步寻找并获取更深层的模块节点

> 模块获取
这是一个深度优先递归过程
在下载一个模块之前, 首先要确定其版本, 因为 package.json 写的是 semantic version (semver, 语义化的版本名称), 因为没有人能记住那么多准确版本

如果 package.json 中某个包的版本是 ^1.1.0, npm 就会去仓库中查询符合 1.x.x 形式的最新版本的压缩包地址

如果这个压缩包本地有缓存, 就直接拿, 没有就从仓库下载

递归获取这个新模块的依赖

> 模块扁平化
上一步获取到的是一棵完整的依赖树, 其中可能包含大量重复模块

上文提到, 从 npm3 开始加入了一个 dedupe 过程, 会遍历依赖树上所有节点, 将其逐个放到根(宿主)节点下面, 也就是 node_modules 的第一层, 发现重复的就丢弃

重复模块指的是: 模块名相同且 semver (语义版本) 兼容

如果模块名相同但 semver 不兼容, npm 会将一个版本放进根 node_modules 中, 另一个留在原依赖树位置 (依赖该模块的模块的 node_modules 中)

> 模块安装
生成 node_modules, 执行模块 install, postinstall 生命周期函数

# 相关
[作为前端工程师你真的知道 npm install 原理么？](https://zhuanlan.zhihu.com/p/128625669)
[与你项目相关的npm知识总结](https://segmentfault.com/a/1190000039289332)
[SemVer](https://semver.org/lang/zh-CN/)