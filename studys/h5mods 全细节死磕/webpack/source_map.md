# 为什么用 eval 包裹 source map?
> 来自: https://github.com/webpack/docs/wiki/build-performance#sourcemaps
> devtool: "eval-source-map" is really as good as devtool: "source-map", but can cache SourceMaps for modules. It's much faster for rebuilds.

猜测的情况是, source-map 无法缓存 sourcemap 的, 因为生成的代码是整体排列下来的, 一旦某一个模块做出修改, 都可能影响到整体文件的行列格式, 而 webpack 又不具备完整的程序流分析能力 (开销高, 也不是很有必要), 所以只能每次重新生成整体的 sourceMap

而用 eval 包裹各个模块的话, 相当于把每个模块的代码单独放在一行里, 并且都能以 DataUrl 的形式为每个模块指定 sourcemap (浏览器的支持?), 这样在 rebuild 的时候对每一个模块进行检查, 如果没有变动就直接用之前的 sourcemap DataUrl, so it is faster

# 为什么要用 base64?
> 来自知乎
> 一些答案其实已经说到了，但是可以更明确一点。Base64编码是从二进制值到某些特定字符的编码，这些特定字符一共64个，所以称作Base64。为什么不直接传输二进制呢？比如图片，或者字符，既然实际传输时它们都是二进制字节流。而且即使Base64编码过的字符串最终也是二进制（通常是UTF-8编码，兼容ASCII编码）在网络上传输的，那么用4/3倍带宽传输数据的Base64究竟有什么意义？真正的原因是二进制不兼容。某些二进制值，在一些硬件上，比如在不同的路由器，老电脑上，表示的意义不一样，做的处理也不一样。同样，一些老的软件，网络协议也有类似的问题。但是万幸，Base64使用的64个字符，经ASCII/UTF-8编码后在大多数机器，软件上的行为是一样的。

# source-map 的原理是什么?
首先的首先, sourcemap 这个能力是由浏览器提供的
它所做的就是把两个文件中的字符, 按照某种规则进行了一一对应
对应后的结果, 就是一堆数据结构, 浏览器中的 js 解释器会读取它, 并按照既定的规则去解释这个对应关系
一个基本的 sourcemap 文件大约长这样:
```js
{
  version : 3, // sourcemap 所使用规则的版本号
  file: "bundle.js", // 转换后的文件名
  sourceRoot : "", // 转换前的文件所在目录, 为空表示与转换前文件在同一目录
  sources: ["foo.js", "bar.js"], // 转换前的文件名, 数组表示存在多个文件合并
  names: ["source", "maps", "are", "fun"], // 转换前的所有变量和属性名
  mappings: "AAgBC,SAAQ,CAAEA;" // 真正记录位置信息的字符串, 这就是那些个一一对应
}
```
这里面最重要的, 是这个 mappings
mappings 一般将成为一个很长很长的字符串, 它的结构是:
  每个 ; 对应转换后源码的一行
  每个 , 对应转换后源码的一个位置, 什么是位置?
    位置是一个信息集合, 每个位置使用五位, 表示了五个字段
      第一位 表示这个位置在 转换后代码的 第几列 (行在前面用;分隔好了)
      第二位 表示这个位置在 sources 中的哪个文件
      第三位 表示这个位置在 转换前代码 的第几行
      第四位 表示这个位置在 转换前代码 的第几列
      第五位 表示这个位置是 names 属性中的哪个变量 (没有对应可缺省)
    
# VLQ
位置中的每一位, 都是用 [VLQ编码](https://en.wikipedia.org/wiki/Variable-length_quantity) 表示
由于 VLQ 是变长的, 所以每一位可以由多个字符构成
VLQ 编码的优点就是可以精简地表示很大的数值, 比如用一个字符表示一个大数字
VLQ 借用了 base64 的 64 个字符, 但是与 base64 编码并不相同
