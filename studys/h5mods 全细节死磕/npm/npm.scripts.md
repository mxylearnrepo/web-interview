# npm scripts 黑魔法
我们先来系统地了解一下 npm scripts. 
Node.js 在设计 npm 之初, 允许开发者在 package.json 文件中, 通过 scripts 字段来自定义项目的脚本. 比如我们可以在 package.json 中这样使用:
```json
{
	// ...
  "scripts": {
    "build": "node build.js",
    "dev": "node dev.js",
    "test": "node test.js",
  }
  // ...
}
```
对应上述代码, 我们在项目中可以使用命令行执行相关的脚本:
```shell
$ npm run build
$ npm run dev
$ npm run test
```
其中 build.js、dev.js、test.js 三个 Node.js 模块分别对应上面三个命令行执行命令.
这样的设计, 可以方便我们统计和集中维护项目工程化或基建相关的所有脚本/命令, 也可以利用 npm 很多辅助功能, 例如下面几个功能:

  * 使用 npm 钩子，比如pre、post, 对应命令 npm run build 的钩子命令就是: prebuild和 postbuild

  开发者使用npm run build时，会默认自动先执行npm run prebuild再执行npm run build，最后执行npm run postbuild，对此我们可以自定义:
  ```json
  {
    // ...
    "scripts": {
      "prebuild": "node prebuild.js",
      "build": "node build.js",
      "postbuild": "node postbuild.js",
    }
    // ...
  }
  ```
  还有一个比较常见的 postinstall 钩子, 会在 npm install 后执行我们规定的脚本

  * 使用 npm 提供的 process.env.npm_lifecycle_event 等环境变量. 通过process.env.npm_lifecycle_event, 可以在相关 npm scripts 脚本中获得当前运行的脚本名称

  * 使用 npm 提供的npm_package_能力, 获取 package.json 中的相关字段, 比如下面代码:
  ```js
  // 获取 package.json 中的 name 字段值
  console.log(process.env.npm_package_name)

  // 获取 package.json 中的 version 字段值
  console.log(process.env.npm_package_version)
  ```

更多 npm 为 npm scripts 提供的“黑魔法” 可以前往 https://docs.npmjs.com/ 进行了解

# npm scripts 原理
其实 npm scripts 原理比较简单. 我们依靠 npm run xxx 来执行一个 npm scripts, 那么核心奥秘就在于 npm run 了. npm run 会自动创建一个 Shell（实际使用的 Shell 会根据系统平台而不同, 类 UNIX 系统里, 如 macOS 或 Linux 中指代的是 /bin/sh, 在 Windows 中使用的是 cmd.exe）, 我们的 npm scripts 脚本就在这个新创建的 Shell 中被运行. 
这样一来, 我们可以得出几个关键结论:

 - 只要是 Shell 可以运行的命令, 都可以作为 npm scripts 脚本

 - npm 脚本的退出码, 也自然遵守 Shell 脚本规则

 - 如果我们的系统里安装了 Python, 可以将 Python 脚本作为 npm scripts

 - npm scripts 脚本可以使用 Shell 通配符等常规能力

比如这样的代码：
```js
{
  // ...
  "scripts": {
    "lint": "eslint **/*.js",
  }
  // ...
}
```
*表示任意文件名, **表示任意一层子目录, 在执行npm run lint后, 就可以对当前目录下, 任意一层子目录的 js 文件进行 lint 审查

另外, 请你思考 npm run创建出来的 Shell 有什么特别之处呢？

我们知道, node_modules/.bin 子目录中的所有脚本都可以直接以脚本名的形式调用, 而不必写出完整路径, 比如下面代码:
```js
{
	// ...
  "scripts": {
    "build": "webpack", // 不需要写成 "./node_modules/.bin/webpack"
  }
  // ...
}
```
这是为什么呢？
实际上 npm run 创建出来的 Shell 需要将当前目录的 node_modules/.bin 子目录加入 PATH 变量中, 在 npm scripts 执行完成后, 再将 PATH 变量恢复

# npm scripts 技巧
**传递参数**
任何命令都需要进行参数传递, npm scripts 遵循惯例使用 -- 标记参数
**串行并行**
可以通过 && 符号串行执行脚本, 通过 & 符号并行执行
这两种串行/并行执行方式其实是 Bash 的能力, 社区里也封装了很多串行/并行执行脚本的公共包供开发者选用, 比如：npm-run-all 就是一个常用的例子
**git-hooks**
npm scripts 可以和 git-hooks 相结合
可用的工具有 pre-commit, husky, lint-staged 等
**兼容问题**
npm scripts 理论上在任何系统都能 just work
跨平台方案比如: [cross-env](https://www.npmjs.com/package/cross-env)