# 组件
组件 = 
  响应式数据 (Object.defineproperty, Proxy) + 
  模板编译 -> 产出 render 函数 + 
  render 函数 -> 产出虚拟 DOM + 
  逻辑复用 (mixin, 高阶函数, composive api, hooks)

## 组件的分类
函数式组件 与 有状态组件 的区别如下：
 - 函数式组件：
   是一个纯函数
   没有自身状态，只接收外部数据
   产出 VNode 的方式：单纯的函数调用

 - 有状态组件：
   是一个类，可实例化
   可以有自身状态
   产出 VNode 的方式：需要实例化，然后调用其 render 函数
   - 普通的有状态组件
   - 需要被 keepAlive 的有状态组件
   - 已经被 keepAlive 的有状态组件

<br>

## 模板编译与组件化
