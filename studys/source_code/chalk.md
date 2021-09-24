[源码赏析 chalk](https://www.jianshu.com/p/654f06f36ec5)

# 原理
可以通过在一段文本前后添加特殊字符来对其进行颜色或格式自定义

# 能力懒加载
在 chalk 上有很多颜色方法可以直接调用
但是我们肯定一次用不到所有的方法呀, 那么一开始就加载全部功能就有点浪费

chalk 妙用 getter 属性实现了"功能懒加载"
```js
for (const [styleName, style] of Object.entries(ansiStyles)) {
  styles[styleName] = {
    get() {
      const builder = createBuilder(this, createStyler(style.open, style.close, this[STYLER]), this[IS_EMPTY]);
      Object.defineProperty(this, styleName, {value: builder});
      return builder;
    }
  };
}
```
一个漂亮的解构循环语法, 然后在 get 方法中给当前这个 style 重新 define 了一个 builder (不用管它是啥) 作为其新值
这样只有到之后第一次用到这个 style 了 (即判断发现有 get 方法) 才会第一次调用 get 方法生成这个 builder 作为其值, 然后这个 get 就被丢弃了
以后再访问该 style 直接得到的就是这个 builder, 上面这个 get 方法也可以替换为 Object.defineProperty 定义的 getter