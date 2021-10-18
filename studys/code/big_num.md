[JS 实现两个大数相加？](https://zhuanlan.zhihu.com/p/72179476)

let sum = a + b
Js 整数运算超过 安全范围 就会损失精度
所以要拿 字符串 表示巨大数字

假如我们要进行 9007199254740991 + 1234567899999999999
```js
let a = "9007199254740991";
let b = "1234567899999999999";

function add(a ,b) {
  let maxLen = Math.max(a.length, b.length)
  // 用 0 补齐短的
  a = a.padStart(maxLen, 0)
  b = b.padStart(maxLen, 0)
  // 从个位开始相加
  function add(a, b) {
    let t = 0 // 求和
    let c = 0 // 进位
    let sum = ''
    for (let i = maxLen - 1; i >= 0; --i) {
      t = parseInt(a[i]) + parseInt(b[i]) + c
      c = Math.floor(t/10) // c 是否只能是 1 0
      sum = t%10 + sum
    }
    if (c === 1) {
      sum = '1' + sum
    }
    return sum
  }
}
```