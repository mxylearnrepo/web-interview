# acorn
A tiny, fast JavaScript parser, written completely in JavaScript

基本用法非常简单:
```js
const acorn = require('acorn')
let code = 1 + 2
console.log(acorn.parse(code)) // an AST tree
```

执行过程:
code => tokenizer (词法分析) => [token, token, token ...] => 语法分析 => AST

acorn 将 词法分析 和 语法分析 交替进行, 只需扫描一遍代码即可得到最终 AST tree
词法分析:
  1. 明确需要分析哪些 Token 类型
     * 关键字 import function return 等
     * 变量名
     * 运算符号
     * 结束标志
  2. 使用状态机输出分词
     简单来讲就是消费 每一个 源码中的字符, 对字符意义进行状态机判断
语法分析: 一段 source code 可以用
 * Program: 整个程序
 * Statement: 语句
 * Expression: 表达式
来描述
最终产出的 AST , 也是这三种元素的数据结构拼合

## acorn 实验
实现一个简单的 Tree Shaking 脚本


