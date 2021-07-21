# 说一下 tree-shaking
tree-shaking 的大前提是 esm 的静态性:
  - 必须出现在 js 文件顶部
  - 模块各个内容必须单独 export 出去 (非常重要)
  - export default 的导出不可以解构 (对 tree-shaking 似乎没什么帮助呢)
  - ...


