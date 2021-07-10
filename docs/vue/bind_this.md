# 面试官：React的事件函数为什么要绑定this？

写 React 组件有一个奇怪的现象:
```js
class Toggle extends React.Component {
  constructor(props) {
    super(props);
    this.state = {isToggleOn: true};

    // This binding is necessary to make `this` work in the callback
    this.handleClick = this.handleClick.bind(this);
  }

  handleClick() {
    this.setState(state => ({
      isToggleOn: !state.isToggleOn
    }));
  }

  render() {
    return (
      <button onClick={this.handleClick}>
        {this.state.isToggleOn ? 'ON' : 'OFF'}
      </button>
    );
  }
}
```
首先回顾一下 [Js this 指向的规律](../JavaScript/this.patch.md)
那么对于上面这个名为 Toggle 的组件来说, 在 render 函数最终变成的 React.createElement 里面, button 的 onClick 属性被赋值成了 this.handleClick, 但是调用时已经丢失了上下文对象, 这个 onClick 既不属于 new 绑定, 也属于隐式绑定, 所以被降级判定为 默认绑定
而严格模式下, 这个默认绑定就会是 undefined, 为了避免这一点, 就需要将它变成一个显式绑定:
```js
	this.handleClick = this.handleClick.bind(this)
```
但是我有一个疑问: 把这个东西交给 JSX 的某个自定义 babel-plugin 去自动完成一下不可以吗? 为什么要把这个东西交给用户处理

类似的问题还有一个编译 JSX 到 React.createElement 的时候, 这里如果用户没有引入 React (虽然用户代码没用到), 就无法执行, 这样合理吗?

## 相关链接

- https://zhuanlan.zhihu.com/p/365770419