## 数组遍历
[查看](./array_api.md)

## 增强循环
**for in**
for in 遍历的是对象的 keys, 而对于数组就是下标
但是使用 for in 遍历得到的顺序是随机的, 因为它本来就是为遍历对象而设计
所以一般不要使用它遍历数组

**for of**
```js
var myArray = [1, 2, 4, 5, 6, 7]
myArray.name = "数组"
myArray.getName = function() { return this.name }
for (var value of myArray) {
  console.log(value)
}
// 不会打印出 name 和 getName
```
用 for of 遍历数组可以使用 break, continue, return 等在 Array.prototype api 下不能用的语句
for of 还可以用于遍历 Map 和 Set 等实现了 Symbol.iterator 方法 的任何对象
for of 的原理就是调用一个对象的 Symbol.iterator 方法, 该方法返回一个迭代器对象, 即任意含有一个 next 方法的对象, 然后 for of 在循环中重复调用这个方法, 直到迭代器对象的 next 返回值中的 done 为 true
一个简单的例子:
```js
var obj = {
  [Symbol.iterator]: function () {
    return this;
  },
  next: function () {
    return {done: false, value: 0};
  }
}
```