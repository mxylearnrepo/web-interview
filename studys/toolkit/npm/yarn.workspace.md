# 启用 yarn workspace
在使用了 yarn 的项目中, 只需要简单的修改项目根目录 package.json
```json
{
  "private": true,
  "workspaces": ["proj-1", "proj-2"]
}
```
接着在 proj-1 和 proj-2 中分别加入 package.json, 里面分别引入不同的依赖
这个时候执行 yarn install, 可以发现所有子项目的依赖全都下载到了根目录的 node_modules 里面
子项目的 node_modules 里只会出现个别依赖的 bin 目录下可执行文件的软链接
同时各子项目的 软链接 也会出现在根目录 node_modules 里, 其原理与 [npm link](./npm.link.md) 相同
这样各个子项目相互之间就可以直接 require 用了

## vs lerna
我们发现 yarn workspace 和lerna 之间有很多共同之处, 解决的问题也有部分重叠
但是 yarn workspace 没有像 lerna 提供了大量 API

事实上 lerna 和 yarn workspace 可以共存, 两者搭配使用能发挥更大作用
主流用法是, lerna 只负责 npm 包管理和发布, workspace 负责子项目依赖管理

