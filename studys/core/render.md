# 字符串模板的渲染
```js
function render(template, data) {
  const reg = /\{\{\w+\}\}/ // 经典的字符串模板正则
  if (reg.test(template)) {
    const key = reg.exec(template)[1] // 从模板里面取出属性名
    const value = key.split('.').reduce((pre, cur) => pre[cur], data)
    template = template.replace(reg, value) // 将第一个模板字符串替换
    return render(template, data) // 用递归代替循环, 继续替换 template 字符串 
  }
}
```