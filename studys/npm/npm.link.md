# npm link
开发完一个 npm 库想验证它, 这时候又不能直接发布到 npm 源上, 
手动复制粘贴到某个项目的 node_modules 中不是不可以, 但是太 low 了

npm link:
假设现在有 packageA 和 projectA, 
packageA 现在更新了, 需要在 projectA 中验证

先在 packageA 的目录下执行 npm link, packageA 目录就被链接到全局

然后在 projectA 中执行 npm lin npm-packageA 命令, 它就会去全局软链接
/usr/local/lib/node_modules/ 这个路径下去找有没有 packageA 这个包

如果有就在 projectA 的 node_moduels 中建立一个 pacakgeA 的软链接

不要忘了调试结束后执行 npm unlink 取消关联

## 进一步
npm link 有两个能力:
  * 为目标 npm 模块创建软链接, 将其目录链接到全局 node 模块安装路径 /usr/local/lib/node_modules/ 中
  * 为目标模块的可执行 bin 文件 (package.json 中定义的 bin 字段) 创建软链接, 将其链接到全局 node 命令安装路径 /usr/local/bin/ 中
